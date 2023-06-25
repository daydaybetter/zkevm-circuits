## State Circuit

### State Circuit Configure

State电路实现了Stack，Memory以及Storage的检查。State电路实现在`zkevm-circuits/zkevm-circuits/src/state_circuit.rs`。

```rust
pub struct StateCircuit<F> {
    /// Rw rows
    pub rows: Vec<Rw>,
    pub(crate) updates: MptUpdates,
    pub(crate) n_rows: usize,
    pub(crate) exports: std::cell::RefCell<Option<StateCircuitExports<Assigned<F>>>>,
    #[cfg(any(feature = "test", test, feature = "test-circuits"))]
    overrides: HashMap<(dev::AdviceColumn, isize), F>,
    _marker: PhantomData<F>,
}
```

StateCircuit的configure逻辑由StateCircuitConfig的new函数实现。

```rust
impl<F: Field> Circuit<F> for StateCircuit<F>
where
    F: Field,
{
    ...

    fn configure(meta: &mut ConstraintSystem<F>) -> Self::Config {
        let rw_table = RwTable::construct(meta);
        let mpt_table = MptTable::construct(meta);
        let challenges = Challenges::construct(meta);

        let config = {
            let challenges = challenges.exprs(meta);
            StateCircuitConfig::new(
                meta,
                StateCircuitConfigArgs {
                    rw_table,
                    mpt_table,
                    challenges,
                },
            )
        };

        (config, challenges)
    }

    ...
}
```

```rust
impl<F: Field> SubCircuitConfig<F> for StateCircuitConfig<F> {
    ...

    /// Return a new StateCircuitConfig
    fn new(
        meta: &mut ConstraintSystem<F>,
        Self::ConfigArgs {
            rw_table,
            mpt_table,
            challenges,
        }: Self::ConfigArgs,
    ) -> Self {
        let selector = rw_table.q_enable;
        log::debug!("state circuit selector {:?}", selector);
        let lookups = LookupsChip::configure(meta);
        let power_of_randomness: [Expression<F>; 31] = challenges.evm_word_powers_of_randomness();

        let rw_counter = MpiChip::configure(meta, selector, rw_table.rw_counter, lookups);
        let tag = BinaryNumberChip::configure(meta, selector, Some(rw_table.tag.into()));
        let id = MpiChip::configure(meta, selector, rw_table.id, lookups);
        let address = MpiChip::configure(meta, selector, rw_table.address, lookups);

        let storage_key = RlcChip::configure(
            meta,
            selector,
            rw_table.storage_key,
            lookups,
            challenges.evm_word(),
        );

        let initial_value = meta.advice_column_in(SecondPhase);
        let is_non_exist = BatchedIsZeroChip::configure(
            meta,
            (SecondPhase, SecondPhase),
            |meta| meta.query_fixed(selector, Rotation::cur()),
            |meta| {
                [
                    meta.query_advice(rw_table.field_tag, Rotation::cur()),
                    meta.query_advice(initial_value, Rotation::cur()),
                    meta.query_advice(rw_table.value, Rotation::cur()),
                ]
            },
        );
        let mpt_proof_type = meta.advice_column_in(SecondPhase);
        let state_root = meta.advice_column_in(SecondPhase);
        meta.enable_equality(state_root);

        let sort_keys = SortKeysConfig {
            tag,
            id,
            field_tag: rw_table.field_tag,
            address,
            storage_key,
            rw_counter,
        };

        let lexicographic_ordering = LexicographicOrderingConfig::configure(
            meta,
            sort_keys,
            lookups,
            power_of_randomness.clone(),
        );

        // annotate columns
        rw_table.annotate_columns(meta);
        mpt_table.annotate_columns(meta);

        let config = Self {
            selector,
            sort_keys,
            initial_value,
            is_non_exist,
            mpt_proof_type,
            state_root,
            lexicographic_ordering,
            not_first_access: meta.advice_column(),
            lookups,
            power_of_randomness,
            rw_table,
            mpt_table,
        };

        let mut constraint_builder = ConstraintBuilder::new();
        meta.create_gate("state circuit constraints", |meta| {
            let queries = queries(meta, &config);
            constraint_builder.build(&queries);
            constraint_builder.gate(queries.selector)
        });
        for (name, lookup) in constraint_builder.lookups() {
            meta.lookup_any(name, |_| lookup);
        }

        config
    }

    ...
}
```

涉及到的tag：Start、Memory、Stack、AccountStorage、TxAccessListAccount、TxAccessListAccountStorage、TxRefund、Account、CallContext、TxLog
<font color="red">对比规范，少了TxReceipt这种tag</font>

#### 约束

具体约束参考zkEVM规范中StateProof的[电路约束](https://arsyun.yuque.com/bp07hf/usehqy/wwb8glzy14tmgnbe#47439aad)部分。

### State Circuit Assign

所有的witness信息都存储在rw_map变量中。在过滤出"Memory/Stack/AccountStorage"信息后，按照key，rw_counter的顺序进行排序。针对每个row的信息，通过assign_row函数设置所有row信息。

```rust
pub fn assign(
    &self,
    layouter: &mut impl Layouter<F>,
    rows: &[Rw],
    updates: &MptUpdates,
    n_rows: usize, // 0 means dynamically calculated from `rows`.
    challenges: &Challenges<Value<F>>,
) -> Result<(), Error> {
    layouter.assign_region(
        || "state circuit",
        |mut region| {
            self.assign_with_region(&mut region, rows, updates, n_rows, challenges.evm_word())
        },
    )?;
    Ok(())
}

fn assign_with_region(
    &self,
    region: &mut Region<'_, F>,
    rows: &[Rw],
    updates: &MptUpdates,
    n_rows: usize, // 0 means dynamically calculated from `rows`.
    randomness: Value<F>,
) -> Result<StateCircuitExports<Assigned<F>>, Error> {
    let tag_chip = BinaryNumberChip::construct(self.sort_keys.tag);

    let (rows, padding_length) = RwMap::table_assignments_prepad(rows, n_rows);
    log::info!(
        "state circuit assign total rows {}, n_rows {}, padding_length {}",
        rows.len(),
        n_rows,
        padding_length
    );
    let rows_len = rows.len();

    let mut state_root =
        randomness.map(|randomness| rlc::value(&updates.old_root().to_le_bytes(), randomness));

    let mut start_state_root: Option<AssignedCell<_, F>> = None;
    let mut end_state_root: Option<AssignedCell<_, F>> = None;
    // annotate columns
    self.annotate_circuit_in_region(region);

    for (offset, row) in rows.iter().enumerate() {
        if offset == 0 || offset + 1 >= padding_length {
            log::trace!("state circuit assign offset:{} row:{:?}", offset, row);
        }

        region.assign_fixed(
            || "selector",
            self.selector,
            offset,
            || Value::known(F::one()),
        )?;

        tag_chip.assign(region, offset, &row.tag())?;

        self.sort_keys
            .rw_counter
            .assign(region, offset, row.rw_counter() as u32)?;

        if let Some(id) = row.id() {
            self.sort_keys.id.assign(region, offset, id as u32)?;
        }

        if let Some(address) = row.address() {
            self.sort_keys.address.assign(region, offset, address)?;
        }

        if let Some(storage_key) = row.storage_key() {
            self.sort_keys
                .storage_key
                .assign(region, offset, randomness, storage_key)?;
        }

        if offset > 0 {
            let prev_row = &rows[offset - 1];
            let index = self
                .lexicographic_ordering
                .assign(region, offset, row, prev_row)?;
            let is_first_access =
                !matches!(index, LimbIndex::RwCounter0 | LimbIndex::RwCounter1);

            region.assign_advice(
                || "not_first_access",
                self.not_first_access,
                offset,
                || Value::known(if is_first_access { F::zero() } else { F::one() }),
            )?;

            if is_first_access {
                // If previous row was a last access, we need to update the state root.
                state_root = randomness
                    .zip(state_root)
                    .map(|(randomness, mut state_root)| {
                        if let Some(update) = updates.get(prev_row) {
                            let (new_root, old_root) = update.root_assignments(randomness);
                            assert_eq!(state_root, old_root);
                            state_root = new_root;
                        }
                        if matches!(row.tag(), RwTableTag::CallContext)
                            && !row.is_write()
                            && row.value_assignment(randomness) != F::zero()
                        {
                            log::error!("invalid call context: {:?}", row);
                        }
                        state_root
                    });
            }
        }

        // The initial value can be determined from the mpt updates or is 0.
        let initial_value = randomness.map(|randomness| {
            updates
                .get(row)
                .map(|u| u.value_assignments(randomness).1)
                .unwrap_or_default()
        });
        region.assign_advice(
            || "initial_value",
            self.initial_value,
            offset,
            || initial_value,
        )?;

        // Identify non-existing if both committed value and new value are zero.
        let is_non_exist_inputs = randomness.map(|randomness| {
            let (_, committed_value) = updates
                .get(row)
                .map(|u| u.value_assignments(randomness))
                .unwrap_or_default();
            let value = row.value_assignment(randomness);
            [
                F::from(row.field_tag().unwrap_or_default()),
                committed_value,
                value,
            ]
        });
        BatchedIsZeroChip::construct(self.is_non_exist.clone()).assign(
            region,
            offset,
            is_non_exist_inputs,
        )?;
        let mpt_proof_type = is_non_exist_inputs.map(|[_field_tag, committed_value, value]| {
            F::from(match row {
                Rw::AccountStorage { .. } => {
                    if committed_value.is_zero_vartime() && value.is_zero_vartime() {
                        MPTProofType::NonExistingStorageProof as u64
                    } else {
                        MPTProofType::StorageMod as u64
                    }
                }
                Rw::Account { field_tag, .. } => {
                    if committed_value.is_zero_vartime()
                        && value.is_zero_vartime()
                        && matches!(field_tag, AccountFieldTag::CodeHash)
                    {
                        MPTProofType::NonExistingAccountProof as u64
                    } else {
                        *field_tag as u64
                    }
                }
                _ => 0,
            })
        });
        region.assign_advice(
            || "mpt_proof_type",
            self.mpt_proof_type,
            offset,
            || mpt_proof_type,
        )?;

        // TODO: Switch from Rw::Start -> Rw::Padding to simplify this logic.
        // State root assignment is at previous row (offset - 1) because the state root
        // changes on the last access row.
        if offset != 0 {
            let assigned = region.assign_advice(
                || "state_root",
                self.state_root,
                offset - 1,
                || state_root,
            )?;
            if start_state_root.is_none() {
                start_state_root.replace(assigned);
            }
        }

        if offset + 1 == rows_len {
            // The last row is always a last access, so we need to handle the case where the
            // state root changes because of an mpt lookup on the last row.
            if let Some(update) = updates.get(row) {
                state_root = randomness.zip(state_root).map(|(randomness, state_root)| {
                    let (new_root, old_root) = update.root_assignments(randomness);
                    if !state_root.is_zero_vartime() {
                        assert_eq!(state_root, old_root);
                    }
                    new_root
                });
            }
            let assigned = region.assign_advice(
                || "last row state_root",
                self.state_root,
                offset,
                || state_root,
            )?;
            end_state_root.replace(assigned);
        }
    }

    let start_state_root = start_state_root.expect("should be assigned");
    let end_state_root = end_state_root.expect("should be assigned");
    Ok(StateCircuitExports {
        start_state_root: (start_state_root.cell(), start_state_root.value_field()),
        end_state_root: (end_state_root.cell(), end_state_root.value_field()),
    })
}
```

### EVM/State Circuit证明了什么？

EVM/State Circuit证明了一些指令的正确逻辑执行，并且Memory/Stack的读写访问合理正确。
