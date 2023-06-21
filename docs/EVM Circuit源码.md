## EVM Circuit

关于EVM Circuit各个Column的设计，每种Opcode如何约束以及多个Opcode之间是如何约束以及组合。

从Column的角度来看，EVM Circuit分为三部分：
1. Step Selector（包括当前Step，第一个Step，最后一个Step等等）
2. Step电路 
3. Fixed Table（固定的查找表）
Step电路逻辑是核心部分。所谓的Step是指从电路约束角度的执行的一个步骤。这一部分又分为两部分：a. execution state（Step State Selector） b. 各种执行状态的约束逻辑。

理解EVM Circuit从Configure和Assign入手。

### EVM Circuit Configure

EVM Circuit实现是在`zkevm-circuits/zkevm-circuits/src/evm_circuit.rs`中。

```rust
fn configure(meta: &mut ConstraintSystem<F>) -> Self::Config {
    let challenges = Challenges::construct(meta);
    let challenges_expr = challenges.exprs(meta);
    let rw_table = RwTable::construct(meta);
    let tx_table = TxTable::construct(meta);
    let bytecode_table = BytecodeTable::construct(meta);
    let block_table = BlockTable::construct(meta);
    let q_copy_table = meta.fixed_column();
    let copy_table = CopyTable::construct(meta, q_copy_table);
    let keccak_table = KeccakTable::construct(meta);
    let exp_table = ExpTable::construct(meta);
    (
        EvmCircuitConfig::new(
            meta,
            EvmCircuitConfigArgs {
                challenges: challenges_expr,
                tx_table,
                rw_table,
                bytecode_table,
                block_table,
                copy_table,
                keccak_table,
                exp_table,
            },
        ),
        challenges,
    )
}
```

EVM Circuit的电路约束由两部分组成：1. fixed_table 2. Execution 部分。fixed_table是一些固定的表信息，占用4个column，<font color="red">分别对应tag/value1/value2/结果</font>。

tag种类为17种，定义在`zkevm-circuits/zkevm-circuits/src/evm_circuit/table.rs`中。

```rust
pub enum FixedTableTag {
    Zero = 0,
    Range5,
    Range16,
    Range32,
    Range64,
    Range128,
    Range256,
    Range512,
    Range1024,
    SignByte,
    BitwiseAnd,
    BitwiseOr,
    BitwiseXor,
    ResponsibleOpcode,
    Pow2,
    ConstantGasCost,
    PrecompileInfo,
}
```

接着查看Execution的部分。ExecutionConfig的configure函数定义了电路的其他约束：

```rust
pub(crate) fn configure(
    meta: &mut ConstraintSystem<F>,
    challenges: Challenges<Expression<F>>,
    fixed_table: &dyn LookupTable<F>,
    byte_table: &dyn LookupTable<F>,
    tx_table: &dyn LookupTable<F>,
    rw_table: &dyn LookupTable<F>,
    bytecode_table: &dyn LookupTable<F>,
    block_table: &dyn LookupTable<F>,
    copy_table: &dyn LookupTable<F>,
    keccak_table: &dyn LookupTable<F>,
    exp_table: &dyn LookupTable<F>,
) -> Self {
    ...
}
```

q_step：Step的Selector
num_rows_until_next_step：
q_step_first：第一个Step的Selector
q_step_last：最后一个Step的Selector
<font color="red">对于一个Step，采用32个Column的数据进行约束</font>

#### Step

Step可以理解是“一个步骤”执行。从电路角度来看，Step是一个由140列21行的电路构成。Step的参数信息定义在`zkevm-circuits/zkevm-circuits/src/evm_circuit/param.rs`中。

```rust
// Step dimension
pub(crate) const STEP_WIDTH: usize = 140;
/// Step height
pub const MAX_STEP_HEIGHT: usize = 21;
```

Step定义在`zkevm-circuits/zkevm-circuits/src/evm_circuit/step.rs`中。

```rust
pub(crate) struct Step<F> {
    pub(crate) state: StepState<F>,
    pub(crate) cell_manager: CellManager<F>,
}
```

Step由StepState和CellManager组成。CellManager定义在`zkevm-circuits/zkevm-circuits/src/evm_circuit/step.rs`中，包含了一个Step涉及的所有信息。
```rust
pub(crate) struct CellManager<F> {
    width: usize,
    height: usize,
    cells: Vec<Cell<F>>,
    columns: Vec<CellColumn<F>>,
}
```

StepState数据结构包括了一个Step对应的状态信息：
```rust
pub(crate) struct StepState<F> {
    /// The execution state selector for the step
    pub(crate) execution_state: DynamicSelectorHalf<F>,
    /// The Read/Write counter
    pub(crate) rw_counter: Cell<F>,
    /// The unique identifier of call in the whole proof, using the
    /// `rw_counter` at the call step.
    pub(crate) call_id: Cell<F>,
    /// The transaction id of this transaction within the block.
    pub(crate) tx_id: Cell<F>,
    /// Whether the call is root call
    pub(crate) is_root: Cell<F>,
    /// Whether the call is a create call
    pub(crate) is_create: Cell<F>,
    /// The block number the state currently is in. This is particularly
    /// important as multiple blocks can be assigned and proven in a single
    /// circuit instance.
    pub(crate) block_number: Cell<F>,
    /// Denotes the hash of the bytecode for the current call.
    /// In the case of a contract creation root call, this denotes the hash of
    /// the tx calldata.
    /// In the case of a contract creation internal call, this denotes the hash
    /// of the chunk of bytes from caller's memory that represent the
    /// contract init code.
    pub(crate) code_hash: Cell<F>,
    /// The program counter
    pub(crate) program_counter: Cell<F>,
    /// The stack pointer
    pub(crate) stack_pointer: Cell<F>,
    /// The amount of gas left
    pub(crate) gas_left: Cell<F>,
    /// Memory size in words (32 bytes)
    pub(crate) memory_word_size: Cell<F>,
    /// The counter for reversible writes
    pub(crate) reversible_write_counter: Cell<F>,
    /// The counter for log index
    pub(crate) log_id: Cell<F>,
}
```

StepState的各个变量的定义：

- execution_state：表明当前Step的执行状态。一个Step的执行状态定义在step.rs。
- rw_counter - Stack/Memory访问时采用rw_counter区分开同一个地址的访问
- call_id - 每一个函数调用赋予一个id，用于区分不同的程序
- is_root - 是否是根
- is_create - 是否是create调用
- code_source - 调用程序的标示（一般是个程序的hash结果）
- program_counter - PC
- stack_point - 栈指针
- gas_left - 剩下的gas数量
- memory_word_size - memory的大小(以word（32字节）为单位)
- state_write_counter - 状态写入的计数器

```rust
pub enum ExecutionState {
    // Internal state
    BeginTx,
    EndTx,
    EndInnerBlock,
    EndBlock,
    // Opcode successful cases
    STOP,
    ADD_SUB,     // ADD, SUB
    MUL_DIV_MOD, // MUL, DIV, MOD
    SDIV_SMOD,   // SDIV, SMOD
    SHL_SHR,     // SHL, SHR
    ADDMOD,
    MULMOD,
    EXP,
    SIGNEXTEND,
    CMP,  // LT, GT, EQ
    SCMP, // SLT, SGT
    ISZERO,
    BITWISE, // AND, OR, XOR
    NOT,
    BYTE,
    SAR,
    SHA3,
    ADDRESS,
    BALANCE,
    ORIGIN,
    CALLER,
    CALLVALUE,
    CALLDATALOAD,
    CALLDATASIZE,
    CALLDATACOPY,
    CODESIZE,
    CODECOPY,
    GASPRICE,
    EXTCODESIZE,
    EXTCODECOPY,
    RETURNDATASIZE,
    RETURNDATACOPY,
    EXTCODEHASH,
    BLOCKHASH,
    BLOCKCTXU64,  // TIMESTAMP, NUMBER, GASLIMIT
    BLOCKCTXU160, // COINBASE
    BLOCKCTXU256, // DIFFICULTY, BASEFEE
    CHAINID,
    SELFBALANCE,
    POP,
    MEMORY, // MLOAD, MSTORE, MSTORE8
    SLOAD,
    SSTORE,
    JUMP,
    JUMPI,
    PC,
    MSIZE,
    GAS,
    JUMPDEST,
    PUSH, // PUSH0, PUSH1, PUSH2, ..., PUSH32
    DUP,  // DUP1, DUP2, ..., DUP16
    SWAP, // SWAP1, SWAP2, ..., SWAP16
    LOG,  // LOG0, LOG1, ..., LOG4
    CREATE,
    CREATE2,
    CALL_OP,       // CALL, CALLCODE, DELEGATECALL, STATICCALL
    RETURN_REVERT, // RETURN, REVERT
    SELFDESTRUCT,
    // Error cases
    ErrorInvalidOpcode,
    ErrorStack,
    ErrorWriteProtection,
    ErrorInvalidCreationCode,
    ErrorInvalidJump,
    ErrorReturnDataOutOfBound,
    ErrorPrecompileFailed,
    ErrorOutOfGasConstant,
    ErrorOutOfGasStaticMemoryExpansion,
    ErrorOutOfGasDynamicMemoryExpansion,
    ErrorOutOfGasMemoryCopy,
    ErrorOutOfGasAccountAccess,
    // error for CodeStoreOOG and MaxCodeSizeExceeded
    ErrorCodeStore,
    ErrorOutOfGasLOG,
    ErrorOutOfGasEXP,
    ErrorOutOfGasSHA3,
    ErrorOutOfGasCall,
    ErrorOutOfGasSloadSstore,
    ErrorOutOfGasCREATE,
    ErrorOutOfGasSELFDESTRUCT,
    // Precompiles
    PrecompileEcRecover,
    PrecompileSha256,
    PrecompileRipemd160,
    PrecompileIdentity,
    PrecompileBigModExp,
    PrecompileBn256Add,
    PrecompileBn256ScalarMul,
    PrecompileBn256Pairing,
    PrecompileBlake2f,
}
```

一个Step的执行状态，包括了内部的状态（一个区块中的交易，通过BeginTx、EndTx隔离），opcode的成功执行状态以及错误状态。有关opcode的成功执行状态，表示的是一个约束能表示的情况下的执行状态。即，多个opcode，如果采用的同一个约束，可以用一个执行状态表示。举个例子，ADD/SUB opcode采用的是一个约束，对于这两个opcode，可以采用同一个执行状态进行约束。

Step的创建函数（new），在创建Step的时候分别创建了StepState和CellManager。

```rust
pub(crate) fn new(
    meta: &mut ConstraintSystem<F>,
    advices: [Column<Advice>; STEP_WIDTH],
    offset: usize,
    is_next: bool,
) -> Self {
    let height = if is_next {
        STEP_STATE_HEIGHT // Query only the state of the next step.
    } else {
        MAX_STEP_HEIGHT // Query the entire current step.
    };
    let mut cell_manager = CellManager::new(meta, height, &advices, offset);
    let state = {
        StepState {
            execution_state: DynamicSelectorHalf::new(
                &mut cell_manager,
                ExecutionState::amount(),
            ),
            rw_counter: cell_manager.query_cell(CellType::StoragePhase1),
            call_id: cell_manager.query_cell(CellType::StoragePhase1),
            tx_id: cell_manager.query_cell(CellType::StoragePhase1),
            is_root: cell_manager.query_cell(CellType::StoragePhase1),
            is_create: cell_manager.query_cell(CellType::StoragePhase1),
            code_hash: cell_manager.query_cell(CellType::StoragePhase2),
            block_number: cell_manager.query_cell(CellType::StoragePhase1),
            program_counter: cell_manager.query_cell(CellType::StoragePhase1),
            stack_pointer: cell_manager.query_cell(CellType::StoragePhase1),
            gas_left: cell_manager.query_cell(CellType::StoragePhase1),
            memory_word_size: cell_manager.query_cell(CellType::StoragePhase1),
            reversible_write_counter: cell_manager.query_cell(CellType::StoragePhase1),
            log_id: cell_manager.query_cell(CellType::StoragePhase1),
        }
    };
    Self {
        state,
        cell_manager,
    }
}
```

按照21行*140列，创建出对应于advice Column的Cell。在同一个Column中不同的Cell，采用Rotation（偏移）进行区分。在编写电路业务的时候，特别注意对Cell的抽象和描述，这些模块化的电路可能会应用多个实例。再观察这些StepState对应的Cell，这些Cell再配合上q_step相应的选择子，就可以精确的描述一个Step的电路约束逻辑。

#### Custom Gate约束

Custom Gate的约束相对来说最复杂。Custom Gate的约束包括：
a. 每一个Step中的target_pair只有一个有效状态。
b. 每一个Step中target_pairs和target_odd是布尔值。检查的方法采用x*(1-x)。
c. 相邻的两个ExecutionState满足约定的条件：如 EndTx 状态只能过渡到 BeginTx 或 EndInnerBlock等。特别注意这些相邻的约束对于最后一个Step不需要满足。
d. 第一个Step的状态必须是BeginTx或者EndBlock，最后一个Step的状态必须是EndBlock。

特别注意，以上custom gate的约束都是相对于某个Step，所以所有约束都必须加上q_step的限制。

#### Advice Column的数据范围约束

约束每个advice都是byte（即Range256）的范围内。因为范围（Range）的约束实现是在fixed_table中，由4个fixed column组成。所以Range256的范围约束由tag/value等四部分对应。

划分了Colum的类型，定义了每个Step的电路范围，并且约束了每个Step中的状态以及相邻两个Step的状态关系。

#### Gadget约束

对于每一种类型的操作（opcode），创建对应的Gadget约束，具体逻辑实现是在`zkevm-circuits/zkevm-circuits/src/evm_circuit/execution.rs`的configure_gadget函数中：

```rust
fn configure_gadget<G: ExecutionGadget<F>>(
    meta: &mut ConstraintSystem<F>,
    advices: [Column<Advice>; STEP_WIDTH],
    q_usable: Selector,
    q_step: Column<Advice>,
    num_rows_until_next_step: Column<Advice>,
    q_step_first: Selector,
    q_step_last: Selector,
    challenges: &Challenges<Expression<F>>,
    step_curr: &Step<F>,
    height_map: &mut HashMap<ExecutionState, usize>,
    stored_expressions_map: &mut HashMap<ExecutionState, Vec<StoredExpression<F>>>,
    instrument: &mut Instrument,
) -> G {
    ...
}
```

对于Gadget的约束，抽象出EVMConstraintBuilder：

```rust
let mut cb = EVMConstraintBuilder::new(
    step_curr.clone(),
    step_next.clone(),
    challenges,
    G::EXECUTION_STATE,
);

let gadget = G::configure(&mut cb);

Self::configure_gadget_impl(
    meta,
    q_usable,
    q_step,
    num_rows_until_next_step,
    q_step_first,
    q_step_last,
    step_curr,
    step_next,
    height_map,
    stored_expressions_map,
    instrument,
    G::NAME,
    G::EXECUTION_STATE,
    height,
    cb,
);
```

通过EVMConstraintBuilder::new创建EVMConstraintBuilder。对于每一种Gadget，调用相应的configure函数，最终调用EVMConstraintBuilder的build函数完成约束配置。

EVMConstraintBuilder定义在`zkevm-circuits/zkevm-circuits/src/evm_circuit/util/constraint_builder.rs`中：

```rust
pub(crate) struct EVMConstraintBuilder<'a, F> {
    pub max_degree: usize, // 最大程度
    pub(crate) curr: Step<F>, // 当前Step
    pub(crate) next: Step<F>, // 下一个Step
    challenges: &'a Challenges<Expression<F>>,
    execution_state: ExecutionState, // 执行状态
    constraints: Constraints<F>, // 约束约束
    rw_counter_offset: Expression<F>, // 读写计数器偏移
    program_counter_offset: usize,// pc指针
    stack_pointer_offset: Expression<F>, // 栈指针
    log_id_offset: usize,
    in_next_step: bool,
    conditions: Vec<Expression<F>>,
    constraints_location: ConstraintLocation,
    stored_expressions: Vec<StoredExpression<F>>,
    pub(crate) max_inner_degree: (&'static str, usize),
}
```

##### **1. EVMConstraintBuilder::new**
创建EVMConstraintBuilder对象。

##### **2. G::config**
所有的opcode相关的Gadget相关的代码在`zkevm-circuits/zkevm-circuits/src/evm_circuit/execution/`目录下。目前已经支持<font color="red">三十多</font>种Gadget，也就是支持三十多种execution_state。

使用简单的AddSubGadget介绍相关约束：

```rust
pub(crate) struct AddSubGadget<F> {
    same_context: SameContextGadget<F>,
    add_words: AddWordsGadget<F, 2, false>,
    is_sub: PairSelectGadget<F>,
}
```

AddSubGadget依赖于其他三个Gadget：SameContextGadget（约束Context）、AddWordsGadget（Words相加）、PairSelectGadget（选边，选择加或者减）。通过AddSubGadget的configure函数查看相关的约束。

1. 约定opcode的Cell（通过cb.query_cell）和操作字a/b/c（通过cb.query_word_rlc）
   ```rust
   let opcode = cb.query_cell();

   let a = cb.query_word_rlc();
   let b = cb.query_word_rlc();
   let c = cb.query_word_rlc();
   ```
2. 使用AddWordsGadget构建加法约束（如果是减法也能转化为加法）
   ```rust
   let add_words = AddWordsGadget::construct(cb, [a.clone(), b.clone()], c.clone());
   ```
3. 使用PairSelectGadget确定是opcode是ADD还是SUB，如果是SUB操作则交换a和c的值
   ```rust
   let is_sub = PairSelectGadget::construct(
       cb,
       opcode.expr(),
       OpcodeId::SUB.expr(),
       OpcodeId::ADD.expr(),
   );
   ```
4. 约束对应的Stack的变化，如果是ADD：出栈a和b，入栈c；如果是SUB：出栈c和b，入栈a。
   ```rust
   cb.stack_pop(select::expr(is_sub.expr().0, c.expr(), a.expr()));
   cb.stack_pop(b.expr());
   cb.stack_push(select::expr(is_sub.expr().0, a.expr(), c.expr()));
   ```
5. 使用SameContextGadget约束Context的变化
   ```rust
   let step_state_transition = StepStateTransition {
       rw_counter: Delta(3.expr()),
       program_counter: Delta(1.expr()),
       stack_pointer: Delta(1.expr()),
       gas_left: Delta(-OpcodeId::ADD.constant_gas_cost().expr()),
       ..StepStateTransition::default()
   };
   let same_context = SameContextGadget::construct(cb, opcode, step_state_transition);
   ```

注意所有的约束最后都是保存在变量cb.constraints中。

##### **3. cb.build**

cb.build就是对每个Gadget的约束进行修正和补充。对于当前Step的约束或者查找表，加上execution_state的Selector。

```rust
let exec_state_sel = self.curr.execution_state_selector([self.execution_state]);
let mul_exec_state_sel = |c: Vec<(&'static str, Expression<F>)>| {
    c.into_iter()
        .map(|(name, constraint)| (name, exec_state_sel.clone() * constraint))
        .collect()
};
```

ExecuteState是可能的执行状态，相对于q_step来说，execute_state的相对位置固定。并且execute_state的每个Cell的值必须是布尔类型，有且只有一个执行状态有效。针对每个执行状态，存在相应的Gadget电路。所有的Gadget电路在cb.build的阶段都必须加上execution_state_selector。即，整个EVM Circuit电路逻辑上包括了所有执行状态的约束。


#### Stack约束

AddSubGadget电路约束中，利用cb.stack_pop和cb.stack_push函数实现相应Stack的约束。

```rust
pub(crate) fn stack_pop(&mut self, value: Expression<F>) {
    self.stack_lookup(false.expr(), self.stack_pointer_offset.clone(), value);
    self.stack_pointer_offset = self.stack_pointer_offset.clone() + self.condition_expr();
}

pub(crate) fn stack_push(&mut self, value: Expression<F>) {
    self.stack_pointer_offset = self.stack_pointer_offset.clone() - self.condition_expr();
    self.stack_lookup(true.expr(), self.stack_pointer_offset.expr(), value);
}

pub(crate) fn stack_lookup(
    &mut self,
    is_write: Expression<F>,
    stack_pointer_offset: Expression<F>,
    value: Expression<F>,
) {
    self.rw_lookup(
        "Stack lookup",
        is_write,
        RwTableTag::Stack,
        RwValues::new(
            self.curr.state.call_id.expr(),
            self.curr.state.stack_pointer.expr() + stack_pointer_offset,
            0.expr(),
            0.expr(),
            value,
            0.expr(),
            0.expr(),
            0.expr(),
        ),
    );
}

/// Add a Lookup::Rw and increase the rw_counter_offset, useful in normal
/// cases.
fn rw_lookup(
    &mut self,
    name: &'static str,
    is_write: Expression<F>,
    tag: RwTableTag,
    values: RwValues<F>,
) {
    self.rw_lookup_with_counter(
        name,
        self.curr.state.rw_counter.expr() + self.rw_counter_offset.clone(),
        is_write,
        tag,
        values,
    );
    // Manually constant folding is used here, since halo2 cannot do this
    // automatically. Better error message will be printed during circuit
    // debugging.
    self.rw_counter_offset = match self.condition_expr_opt() {
        None => {
            if let Constant(v) = self.rw_counter_offset {
                Constant(v + F::from(1u64))
            } else {
                self.rw_counter_offset.clone() + 1i32.expr()
            }
        }
        Some(c) => self.rw_counter_offset.clone() + c,
    };
}

/// Add a Lookup::Rw without increasing the rw_counter_offset, which is
/// useful for state reversion or dummy lookup.
fn rw_lookup_with_counter(
    &mut self,
    name: &str,
    counter: Expression<F>,
    is_write: Expression<F>,
    tag: RwTableTag,
    values: RwValues<F>,
) {
    let name = format!("rw lookup {}", name);
    self.add_lookup(
        &name,
        Lookup::Rw {
            counter,
            is_write,
            tag: tag.expr(),
            values,
        },
    );
}
```

stack_pop和stack_push的实现类似，都是调整stack_pointer_offset偏移，并通过stack_lookup约束Stack上stack_pointer_offset对应的值。
核心逻辑是rw_loopup，rw_loopup实现又是基于rw_lookup_with_counter。Stack的访问细化为call_id、Stack的偏移以及rw的次数。在同一个Step State中，对同一个Stack的偏移存在多次读写的可能性，这些可能性的约束需要加上读写的次数。
rw_loopup_with_counter的实现就是将lookup（查找表）通过add_lookup记录在ConstraintBuilder的lookups变量中，为后面查找表的Configure做准备。

除了RW（可读写的）查找表外，目前还支持其他一些查找表类型。Lookup枚举类型定义在`zkevm-circuits/zkevm-circuits/src/evm_circuit/table.rs`：
```rust
pub(crate) enum Lookup<F> {
    /// Lookup to fixed table, which contains several pre-built tables such as
    /// range tables or bitwise tables.
    Fixed {...},
    /// Lookup to tx table, which contains transactions of this block.
    Tx {...},
    /// Lookup to read-write table, which contains read-write access records of
    /// time-aware data.
    Rw {...},
    /// Lookup to bytecode table, which contains all used creation code and
    /// contract code.
    Bytecode {...},
    /// Lookup to block table, which contains constants of this block.
    Block {...},
    /// Lookup to copy table.
    CopyTable {...},
    /// Lookup to keccak table.
    KeccakTable {...},
    /// Lookup to exponentiation table.
    ExpTable {...},
    /// Conditional lookup enabled by the first element.
    Conditional(Expression<F>, Box<Lookup<F>>),
}
```

Tx查找表约束交易数据，RW查找表约束可读可写的数据访问（Stack/Memory），Bytecode约束执行的程序，Block约束的是区块相关的数据，CopyTable约束的是多个Cell值相等，KeccakTable约束的是Keccak运算，ExpTable约束的是指数化操作。
对于每一种Step，可能存在Stack或者Memory的操作。为了约束这些操作，除了申请和这些操作数据相关的Cell外，还需要指定Stack Pointer。在Stack Pointer锚定的情况下，某个Step内部的Stack的操作可以独立约束。并且，Stack Pointer对应的Cell在Step中的位置可以通过q_step进行锚定。也就是说，从单个Step的角度来说，约束可以独立表达和实现。当然，Stack的约束除了单个Step的约束外，多个Step之间也存在Stack数据一致性和正确性的约束。这些约束由后续[Lookup约束](Lookup约束)进行约束。

#### Context约束

前面主要是单个Execution State的约束实现。而SameContextGadget约束的是多个Execution State之间约束实现。

具体约束实现逻辑在`zkevm-circuits/zkevm-circuits/src/evm_circuit/util/common_gadget.rs`的SameContextGadget construct函数中：

```rust
pub(crate) fn construct(
    cb: &mut EVMConstraintBuilder<F>,
    opcode: Cell<F>,
    step_state_transition: StepStateTransition<F>,
) -> Self {
    cb.opcode_lookup(opcode.expr(), 1.expr());
    cb.add_lookup(
        "Responsible opcode lookup",
        Lookup::Fixed {
            tag: FixedTableTag::ResponsibleOpcode.expr(),
            values: [
                cb.execution_state().as_u64().expr(),
                opcode.expr(),
                0.expr(),
            ],
        },
    );

    // Check gas_left is sufficient
    let sufficient_gas_left = RangeCheckGadget::construct(cb, cb.next.state.gas_left.expr());

    // Do step state transition
    cb.require_step_state_transition(step_state_transition);

    Self {
        opcode,
        sufficient_gas_left,
    }
}
```

实现了以下约束：

1. opcode_lookup：约束pc对应的opcode一致
2. add_lookup：约束opcode和Step中的execution state一致
3. 检查gas是否足够
4. require_step_state_transition：约束状态转换是否一致（涉及两个Step之间的约束）。比如说pc的关系、stack_pointer的关系以及call_id的关系等等。

#### Lookup约束

在每个Execution State的约束准备就绪后，开始处理所有的Lookup的约束。具体实现在`zkevm-circuits/zkevm-circuits/src/evm_circuit/execution.rs`的ExecutionConfig configure函数中：

```rust
Self::configure_lookup(
    meta,
    fixed_table,
    byte_table,
    tx_table,
    rw_table,
    bytecode_table,
    block_table,
    copy_table,
    keccak_table,
    exp_table,
    &challenges,
    &cell_manager,
);
```

实现的逻辑：每个Lookup的约束需要额外加上查找参数。

### EVM Circuit Assign

EVM Circuit通过assign_block对一个Block中的所有的交易进行证明。具体实现在`zkevm-circuits/zkevm-circuits/src/evm_circuit/execution.rs`的ExecutionConfig assign_block函数中：

```rust
pub fn assign_block(
    &self,
    layouter: &mut impl Layouter<F>,
    block: &Block<F>,
    challenges: &Challenges<Value<F>>,
) -> Result<EvmCircuitExports<Assigned<F>>, Error> {
    let mut is_first_time = true;

    layouter.assign_region(
        || "Execution step",
        |mut region| {
            if is_first_time {
                is_first_time = false;
                region.assign_advice(
                    || "step selector",
                    self.q_step,
                    self.get_num_rows_required(block) - 1,
                    || Value::known(F::zero()),
                )?;
                return Ok(());
            }
            let mut offset = 0;

            // Annotate the EVMCircuit columns within it's single region.
            self.annotate_circuit(&mut region);

            self.q_step_first.enable(&mut region, offset)?;

            let dummy_tx = Transaction::default();
            let last_call = block
                .txs
                .last()
                .map(|tx| tx.calls[0].clone())
                .unwrap_or_else(Call::default);
            let end_block_not_last = &block.end_block_not_last;
            let end_block_last = &block.end_block_last;
            // Collect all steps
            let mut steps = block
                .txs
                .iter()
                .flat_map(|tx| {
                    tx.steps
                        .iter()
                        .map(move |step| (tx, &tx.calls[step.call_index], step))
                })
                .chain(std::iter::once((&dummy_tx, &last_call, end_block_not_last)))
                .peekable();

            let evm_rows = block.circuits_params.max_evm_rows;
            let no_padding = evm_rows == 0;

            // part1: assign real steps
            loop {
                let (transaction, call, step) = steps.next().expect("should not be empty");
                let next = steps.peek();
                if next.is_none() {
                    break;
                }
                let height = step.execution_state.get_step_height();

                // Assign the step witness
                if step.execution_state == ExecutionState::EndTx {
                    let mut tx = transaction.clone();
                    tx.call_data.clear();
                    tx.calls.clear();
                    tx.steps.clear();
                    tx.rlp_signed.clear();
                    tx.rlp_unsigned.clear();
                    let total_gas = {
                        let gas_used = tx.gas - step.gas_left;
                        let current_cumulative_gas_used: u64 = if tx.id == 1 {
                            0
                        } else {
                            // first transaction needs TxReceiptFieldTag::COUNT(3) lookups
                            // to tx receipt,
                            // while later transactions need 4 (with one extra cumulative
                            // gas read) lookups
                            let rw = &block.rws[(
                                RwTableTag::TxReceipt,
                                (tx.id - 2) * (TxReceiptFieldTag::COUNT + 1) + 2,
                            )];
                            rw.receipt_value()
                        };
                        current_cumulative_gas_used + gas_used
                    };
                    log::info!(
                        "offset {} tx_num {} total_gas {} assign last step {:?} of tx {:?}",
                        offset,
                        tx.id,
                        total_gas,
                        step,
                        tx
                    );
                }
                self.assign_exec_step(
                    &mut region,
                    offset,
                    block,
                    transaction,
                    call,
                    step,
                    height,
                    next.copied(),
                    challenges,
                )?;

                // q_step logic
                self.assign_q_step(&mut region, offset, height)?;

                offset += height;
            }

            // part2: assign non-last EndBlock steps when padding needed
            if !no_padding {
                let height = ExecutionState::EndBlock.get_step_height();
                debug_assert_eq!(height, 1);
                // 1 for EndBlock(last), 1 for "part 4" cells
                let last_row = evm_rows - 2;
                log::trace!(
                    "assign non-last EndBlock in range [{},{})",
                    offset,
                    last_row
                );
                if offset > last_row {
                    log::error!(
                        "evm circuit row not enough, offset: {}, max_evm_rows: {}",
                        offset,
                        evm_rows
                    );
                    return Err(Error::Synthesis);
                }
                self.assign_same_exec_step_in_range(
                    &mut region,
                    offset,
                    last_row,
                    block,
                    &dummy_tx,
                    &last_call,
                    end_block_not_last,
                    height,
                    challenges,
                )?;

                for row_idx in offset..last_row {
                    self.assign_q_step(&mut region, row_idx, height)?;
                }
                offset = last_row;
            }

            // part3: assign the last EndBlock at offset `evm_rows - 1`
            let height = ExecutionState::EndBlock.get_step_height();
            debug_assert_eq!(height, 1);
            log::trace!("assign last EndBlock at offset {}", offset);
            self.assign_exec_step(
                &mut region,
                offset,
                block,
                &dummy_tx,
                &last_call,
                end_block_last,
                height,
                None,
                challenges,
            )?;
            self.assign_q_step(&mut region, offset, height)?;
            // enable q_step_last
            self.q_step_last.enable(&mut region, offset)?;
            offset += height;

            // part4:
            // These are still referenced (but not used) in next rows
            region.assign_advice(
                || "step height",
                self.num_rows_until_next_step,
                offset,
                || Value::known(F::zero()),
            )?;
            region.assign_advice(
                || "step height inv",
                self.q_step,
                offset,
                || Value::known(F::zero()),
            )?;

            log::debug!("assign for region done at offset {}", offset);
            Ok(())
        },
    )?;

    log::debug!("assign_block done");

    let final_withdraw_root_cell = self
        .end_block_gadget
        .withdraw_root_assigned
        .borrow()
        .expect("withdraw_root cell should has been assigned");

    // sanity check
    let evm_rows = block.circuits_params.max_evm_rows;
    if evm_rows >= 2 {
        assert_eq!(final_withdraw_root_cell.row_offset, evm_rows - 2);
    }

    let withdraw_root_rlc = challenges
        .evm_word()
        .map(|r| rlc::value(&block.withdraw_root.to_le_bytes(), r));

    Ok(EvmCircuitExports {
        withdraw_root: (final_withdraw_root_cell, withdraw_root_rlc.into()),
    })
}
```

一笔交易Tx的一个个步骤(step)存储在steps变量中。针对每个Step，调用assign_exec_step进行assign。