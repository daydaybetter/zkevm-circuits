### 零知识证明 - zkEVM源代码分析（State Circuit）



- [Star Li](https://learnblockchain.cn/people/28)
-  

- 更新于 2022-05-06 08:51
-  

- 阅读 1239

[前一篇文章](https://learnblockchain.cn/article/3959)介绍了zkEVM的EVM Circuit的电路实现细节，接下来继续介绍State Circuit。

zkEVM是零知识证明相对复杂的零知识证明应用，源代码值得反复阅读和学习。

https://github.com/appliedzkp/zkevm-circuits.git

本文中采用的源代码对应的最后一个提交信息如下：

```sql
commit 1ec38f207f150733a90081d3825b4de9c3a0a724 (HEAD -> main)
Author: z2trillion <z2trillion@users.noreply.github.com>
Date:   Thu Mar 24 15:42:09 2022 -0400
```

前一篇文章介绍了zkEVM的EVM Circuit的电路实现细节，接下来继续介绍State Circuit。

[零知识证明 - zkEVM源代码分析(EVM Circuit)](https://learnblockchain.cn/article/3959)

### State Circuit Configure

State电路实现了Stack，Memory以及Storage的检查。State电路实现在zkevm-circuits/src/state_circuit/state.rs。

```rust
pub struct StateCircuit<
  F: FieldExt,
  const SANITY_CHECK: bool,
  const RW_COUNTER_MAX: usize,
  const MEMORY_ADDRESS_MAX: usize,
  const STACK_ADDRESS_MAX: usize,
  const ROWS_MAX: usize,
> {
  /// randomness used in linear combination
  pub randomness: F,
  /// witness for rw map
  pub rw_map: RwMap,
}
```

StateCircuit的configure逻辑由Config的configure函数实现。

```rust
fn configure(meta: &mut ConstraintSystem<F>) -> Self::Config {
  Config::configure(meta)
}
```

StateCircuit的大体电路结构如下图：

![1.png](https://img.learnblockchain.cn/attachments/2022/05/7nJynHPu6274705b7044b.png!/scale/60)

先着重介绍一下keys，占用了5个column，表示Memory，Stack以及Storage的key信息。State Circuit约束大部分都是围绕key-value的约束。为了兼容Memory，Stack以及Storage的Key信息，采用如下的5个Column。

keys[0] - tag

keys[1] - reserved

keys[2] - account address(memory/stack: 0)

keys[3] - address

keys[4] - storage key(memory/stack: 0)

其中key2，key4是给Storage用的。

Storage约束的信息还不够完整。Memory和Stack的约束逻辑类似。这篇文章详细介绍一下Stack的约束实现。在了解约束之前，先介绍Memory/Stack/Storage的信息组织形式：

| key0(tag) | key1 | key2 | key3(address) | key4 | read/write | rw counter | value |
| :-------- | :--- | :--- | :------------ | :--- | :--------- | :--------- | :---- |
| MEM       | 0    | 0    | 0             | 0    | 1          | 0          | 0     |
| MEM       | 0    | 0    | 0             | 0    | 0          | 2          | 0     |
| STACK     | 0    | 0    | 0             | 0    | 1          | 0          | 0     |
| STACK     | 0    | 0    | 0             | 0    | 0          | 2          | 0     |

所有的witness信息，按照key3（address）排序（按照地址顺序）。针对同一个地址的读写根据rw counter进行排序。

#### Tag分类

Tag类型包括了Memory，Stack以及Storage。针对不同的类型，采用不同的约束方式。

```rust
let q_tag_is = |meta: &mut VirtualCells<F>, tag_value: usize| {
  let tag_cur = meta.query_advice(tag, Rotation::cur());
  let all_possible_values = EMPTY_TAG..=STORAGE_TAG;
  generate_lagrange_base_polynomial(tag_cur, tag_value, all_possible_values)
};
let q_memory = |meta: &mut VirtualCells<F>| q_tag_is(meta, MEMORY_TAG);
let q_stack = |meta: &mut VirtualCells<F>| q_tag_is(meta, STACK_TAG);
let q_storage = |meta: &mut VirtualCells<F>| q_tag_is(meta, STORAGE_TAG);
```

q_tag_is是拉格朗日多项式，针对某种Tag输出为1，其他Tag输出为0。q_memory/q_stack/q_storage就分别是这三种多项式。

#### Key关系判定

在约束电路中需要两种两种Key关系的判定：

1/ 前后的key是否一样

```solidity
let key_is_same_with_prev: [IsZeroConfig<F>; 5] = [0, 1, 2, 3, 4].map(|idx| {
  IsZeroChip::configure(
      meta,
      |meta| meta.query_fixed(s_enable, Rotation::cur()),
      |meta| {
          let value_cur = meta.query_advice(keys[idx], Rotation::cur());
          let value_prev = meta.query_advice(keys[idx], Rotation::prev());
          value_cur - value_prev
      },
      keys_diff_inv[idx],
  )
});
```

2/ 是否key是一样

```solidity
let q_all_keys_same = |_meta: &mut VirtualCells<F>| {
  key_is_same_with_prev[0].is_zero_expression.clone()
      * key_is_same_with_prev[1].is_zero_expression.clone()
      * key_is_same_with_prev[2].is_zero_expression.clone()
      * key_is_same_with_prev[3].is_zero_expression.clone()
      * key_is_same_with_prev[4].is_zero_expression.clone()
};
let q_not_all_keys_same = |meta: &mut VirtualCells<F>| one.clone() - q_all_keys_same(meta);
```

#### 通用约束

无论是Memory/Stack，还是Storage，有一些通用的约束：

1/ is_write必须是布尔型

2/ 如果是对同一个“地址”进行读操作，则读的数据必须和前一行的数据相同。前一行数据要不是同一地址的读，要不就是同一地址的写。

```solidity
cb.require_boolean("is_write should be boolean", is_write);

cb.require_zero(
  "if read and keys are same, value should be same with prev",
  q_all_keys_same(meta) * is_read * (value_cur - value_prev),
);

cb.gate(s_enable)
```

#### RWC约束

对一个地址的读写操作，RW计数器是增长的。所谓的增长就是（rw_counter - rw_counter_prev -1) > 0。大于零的判定就是通过lookup。

```solidity
meta.lookup_any("rw counter monotonicity", |meta| {
  let s_enable = meta.query_fixed(s_enable, Rotation::cur());
  let rw_counter_table = meta.query_fixed(rw_counter_table, Rotation::cur());
  let rw_counter_prev = meta.query_advice(rw_counter, Rotation::prev());
  let rw_counter = meta.query_advice(rw_counter, Rotation::cur());

  vec![(
      s_enable * q_all_keys_same(meta)
          * (rw_counter - rw_counter_prev - one.clone()), /*
                                                            * - 1 because it needs to
                                                            *   be strictly monotone */
      rw_counter_table,
  )]
});
```

#### Stack约束

在Stack的访问地址发生变化时，第一个操作必须是写操作。

```solidity
meta.create_gate("Stack operation", |meta| {
  let mut cb = new_cb();

  let s_enable = meta.query_fixed(s_enable, Rotation::cur());
  let is_write = meta.query_advice(is_write, Rotation::cur());
  let q_read = one.clone() - is_write;
  let key2 = meta.query_advice(keys[2], Rotation::cur());
  let key4 = meta.query_advice(keys[4], Rotation::cur());

  cb.require_zero("key2 is 0", key2);
  cb.require_zero("key4 is 0", key4);

  cb.require_zero(
      "if address changes, operation is always a write",
      q_not_all_keys_same(meta) * q_read,
  );
  cb.gate(s_enable * q_stack(meta))
});
```

Stack地址必须在一定的范围内，并且Stack Pointer的差距不能超过1。

```solidity
meta.lookup_any("Stack address in allowed range", |meta| {
  let q_stack = q_stack(meta);
  let address_cur = meta.query_advice(address, Rotation::cur());
  let stack_address_table_zero =
      meta.query_fixed(stack_address_table_zero, Rotation::cur());

  vec![(q_stack * address_cur, stack_address_table_zero)]
});

meta.create_gate("Stack pointer diff be 0 or 1", |meta| {
  let mut cb = new_cb();
  let s_enable = meta.query_fixed(s_enable, Rotation::cur());
  let q_stack = q_stack(meta);
  let tag_is_same_with_prev = key_is_same_with_prev[0].is_zero_expression.clone();
  let call_id_same_with_prev = key_is_same_with_prev[1].is_zero_expression.clone();
  let stack_ptr = meta.query_advice(keys[3], Rotation::cur());
  let stack_ptr_prev = meta.query_advice(keys[3], Rotation::prev());
  cb.require_boolean(
      "stack pointer only increases by 0 or 1",
      stack_ptr - stack_ptr_prev,
  );
  cb.gate(s_enable * q_stack * tag_is_same_with_prev * call_id_same_with_prev)
});
```

至此，Stack的相关约束就完成了。再看看assign的逻辑实现。

### State Circuit Assign

所有的witness信息都存储在rw_map变量中。在过滤出"Memory/Stack/AccountStorage"信息后，按照key，rw_counter的顺序进行排序。针对每个row的信息，通过assign_row函数设置所有row信息。

```solidity
layouter.assign_region(
  || "State operations",
  |mut region| {
      // TODO: a "START_TAG" row should be inserted before all other rows in the final
      // implmentation. Here we start from 1 to prevent some
      // col.prev() problems since blinding rows are unavailable for constaints.
      let mut offset = 1;

      let mut rows: Vec<RwRow<F>> = [
          RwTableTag::Memory,
          RwTableTag::Stack,
          RwTableTag::AccountStorage,
      ]
      .iter()
      .map(|tag| {
          rw_map.0[tag]
              .iter()
              .map(|rw| rw.table_assignment(randomness))
      })
      .flatten()
      .collect();
      rows.sort_by_key(|rw| (rw.tag, rw.key1, rw.key2, rw.key3, rw.key4, rw.rw_counter));

      if rows.len() >= ROWS_MAX {
          panic!("too many storage operations");
      }
      for (index, row) in rows.iter().enumerate() {
          let row_prev = if index == 0 {
              RwRow::default()
          } else {
              rows[index - 1]
          };
          self.assign_row(
              &mut region,
              offset,
              *row,
              row_prev,
              &key_is_same_with_prev_chips,
          )?;
          offset += 1;
      }

      Ok(())
  },
)
```

### EVM/State Circuit证明了什么？

EVM/State Circuit证明了一些指令的正确逻辑执行，并且Memory/Stack的读写访问合理正确。

![2.png](https://img.learnblockchain.cn/attachments/2022/05/9orZReT6627470701da35.png!/scale/60)

zkEVM除了这两部分电路外，还有一些外延的东西没有证明：1/ 执行程序的一致性 2/ Storage状态的正确性。这些后面再接着聊。