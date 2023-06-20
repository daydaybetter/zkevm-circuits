# zkEVM

## 背景

zkEVM利用零知识证明技术（zk-SNARK）证明：
 * 针对智能合约的交易执行，生成交易证明。在L2实现zkRollup支持可编程性。
 * 针对以太坊的每一个区块，生成区块证明。

证明由两部分组成：State proof（状态证明）和EVM proof（EVM执行证明）。在一条交易执行过程中，State包括Storage，Memory以及Stack的状态。
Bus-mapping是设计的基础思想。一般的PC体系结构中，CPU通过总线访问存储（内存/硬盘），也就是计算和存储分开。zkEVM采用的同样的架构思想，状态的变化和指令的执行分开，并且分别由State proof和EVM proof进行证明。

State proof负责证明Bus Mapping信息的一致性和正确性。一致性指的Bus Mapping和State之间的读写一致。正确性指的是Bus Mapping中的读写状态正确。
EVM proof负责证明EVM opcode计算逻辑的执行正确（如果涉及到State的opcode，保证存储相关的操作正确）。

## Bus Mapping

Bus Mapping包括了两种状态，一种是读取老的状态，一种是写回生成新的状态。无论是老的状态和新的状态，Bus Mapping都是“包含”关系。这种“包含”的mapping关系可以通过Plookup算法证明。

存储状态（Storage/Memory/Stack），采用key-value的格式进行表示。key-value对的绑定关系可以实现为一个序列：

```python
def build_mapping():
    keys = [1,3,5]
    values = [2,4,6]
 
    randomness = hash(keys, values)
    
    mappings = []

    for key , value in zip(keys,values):
        mappings.append(key + randomness*value)
    return(mappings)
```

想证明某个key-value对在一些key-value中，可以通过三个Plookup证明实现：
```python
def open_mapping(mappings, keys, values):
    randomness = hash(keys,values)
    index = 1
    # Prover can chose any mapping, key , value
    mapping = plookup(mappings)
    key = plookup(keys)
    value = plookup(values)
    # But it has to satisfy this check
    require(mappings[index] == key[index] + randomness*value[index])
    # with overwhelming probability will not find an invalid mapping.
```

keys和values是State相关的key-value对。mappings是所有key-value对行程的mapping数组。通过三个Plookup证明：
1/ 某个mapping的key在keys中
2/ 某个mapping的value在values中
3/ 某个mapping在mappings中
并且mapping和key-value满足build_mapping建立的关系。如上的这些证明，能够证明某个key-value对是keys和values的mapping中的部分。
在如上的基础上，可以定义出bus_mapping的数据结构：
```python
bus_mapping[global_counter] = {
    mem: [op, key, value], 
    stack: [op, index, value],
    storage: [op, key, value],
    index: opcode,
    call_id: call_id,
    prog_counter: prog_counter
}
```
其中，global_counter是slot的编号。slot是一个证明逻辑的最小单元。一个交易的证明由多条指令组成，每个指令可能由多个slot组成。op是操作类型，逻辑上由读和写组成。prog_counter是pc。所有的bus_mapping中的读取操作可以通过plookup确定是不是“属于”当前的State。读取状态需要和老的存储状态一致，写入状态需要和更新后的存储状态一致。

在Bus mapping的基础上，采用State Proof和EVM Proof证明数据的读写正确以及和opcode的语义一致。

## State proof

State Proof证明了和“存储”相关的读写和Bus Mapping的数据一致。也就是在“总线”上的读写数据的一致性。State Proof根据存储类型分成三种证明。

### Storage

和Storage相关的opcode罗列如下：

| Flag | Opcode                   | Description                                                  |
| ---- | ------------------------ | ------------------------------------------------------------ |
| 0x54 | SLOAD                    | Read a variable                                              |
| 0x55 | SSTORE                   | Write a variable                                             |
| 0xf0 | CREATE                   | Deploy a contract                                            |
| 0x31 | BALANCE                  | Get a user's balance                                         |
| 0xf1 | CALL                     | Call another account                                         |
| 0xf2 | CALLCODE                 | Message call into this account with another account's code   |
| 0xf4 | DELEGATECALL             | Message call into this account with another account's code and persist changes |
| 0xfa | STATICCALL               | Similar to call but changes are not persistent               |
| 0xfd | REVERT                   | Stop execution and  revert storage changes                   |
|      | Get current storage root | Use this to enforce static calls and reverts                 |

相关的opcode分为两种，一种是Storage读，一种是Storage写。比如SLOAD就是从Storage读取数据到Stack中。Storage相关的opcode对应的读写数据在Bus mapping中。注意，State Proof中的Storage证明只单纯证明Storage相关的读写操作是否在bus mapping中。至于读写数据的语义信息是通过EVM proof进行证明的。

### Memory

从State Proof证明的角度看，所有的内存操作按照index进行排序。index是内存地址。举个例子：

| index | global_counter | value | memory_flag | comment   |
| ----- | -------------- | ----- | ----------- | --------- |
| 0     | 0              | 0     | 1           | init at 0 |
| 0     | 12             | 12    | 1           | write 12  |
| 0     | 24             | 12    | 0           | read 12   |
| 1     | 0              | 0     | 1           | init at 0 |
| 1     | 17             | 32    | 1           | write 32  |
| 1     | 89             | 32    | 0           | read 32   |

地址0和地址1的相关内存操作罗列在一起。每个内存地址在程序开始执行时会初始化为0。也就是global_counter为零时，index 0/1初始为0。从Bus Mapping的角度看，这些内存操作如下：

| global_counter | memory_flag | memory_index | memory_value |
| -------------- | ----------- | ------------ | ------------ |
| 12             | 1           | 0            | 12           |
| 17             | 1           | 1            | 32           |
| 24             | 0           | 0            | 12           |
| 89             | 0           | 1            | 32           |

对于每个地址上的读写数据都需要保持一致性。这些一致性由如下的约束检查：
```python
for i in range(10000):
    # check the rest

    for value, memory_flag, global_counter, index in zip (value, memory_flag, global_counter, index):

        # start  ======
        # binary constraint
        require(memory_flag == 1 or memory_flage == 0)
        # end    ======

        # start2 ======
        # Check the value is corrent
        # if memory_flag == 0 its a read we check
        # previous_value == current value read
        if index != prev_index:
            ## init to zero
            prev_index = index
            require(value == 0)
            require(memory_flag == 1)
        
        if (memory_flag == 0):
            require(current_value == value)
        else (memory_flag == 1):
            # if its a write we just set the current_value = new_value
            current_value = value
        # end2   ======

        # Check global_counter is ordered correctly
        require(global_counter > previous_global_counter)
        previous_global_counter = global_counter

        # start3 ======
        if (global_counter != 0):
            # add to bus mapping
            global_counter = plookup(global_counter, valid_glbal_counters)
            mapping = plookup(mapping, valid_mappings)
            memory = "memory"
            memory_flag = memory_flage
            index = index
            value = value
            # check the key-value mapping
            require(mapping == global_counter + rand[0]*memory + rand[1]*memory_flag+rand[2]*index)
        # end3   ======
```

蓝色部分限制内存只有读写操作，褐色部分限制每个地址内存初始为零，并且读写数据一致，橘黄色部分限制相关的内存操作在bus mapping中。

### Stack

Stack的操作主要分为三类：Push/Pop，Dup和Swap。用1024大小的数组模拟Stack实现。Stack的位置信息的检查由EVM proof完成。Stack的数据和bus mapping的关系由Stack proof完成。基本思路和Memory一致。

## EVM Proof

EVM Proof是EVM执行相关约束的核心。EVM proof需要证明如下的一些逻辑：
1/ EVM执行的代码是指定的transaction的代码逻辑
2/ 和存储相关的语义是否正确，比如Stack的位置管理，SLoad数据和Stack的数据是否一致等等
3/ 和计算相关的语义是否正确，比如算术计算等等
4/ 跳转逻辑是否正确，比如Call指令
5/ Gas消耗计算正确
6/ 程序终止是否正确

Slot是电路的基本单元。多个Slot可以组合在一起实现一个op code的语义。

### 算术计算

以加法为例，通过Plookup可以证明8bit整数的计算。通过多个8bit的加法以及进位，可以实现任意长度的加法。

```python
carry_bit = 0
for i in range(len(value1)):
    result, carry_bit = plookup_add(value[i], value2[i], carry_bit)
    require(result == value2[i])
```

两个数的比较的实现原理和加法类似。

### 跳转逻辑

call_id记录一个执行环境。为了绑定和跳转逻辑的关系，call_id采用如下的计算逻辑：

```
call_id -> rlc(calldata_callid, call_data_start_addr, call_data_size, return_data_callid,return_data_start_addr, return_data_size, origin, caller, call_value, call_stack_depth)
```

rlc是random linear combination。

EVM proof的逻辑相对来说比较复杂。当前公布的文档没有展开描述设计的细节。zkEVM给出了大致的设计思路，细节的部分还需要进一步细化。


# 源码阅读

zkEVM的电路主要由两部分电路组成：EVM Circuit和State Circuit。

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

对于每一种类型的操作（opcode），创建对应的Gadget约束，具体逻辑实现是在`zkevm-circuits/src/evm_circuit/execution.rs`的configure_gadget函数中：

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

# 杂 - other

qL · a + qR · b + qM · a · b + qO · c + qC = 0
a, b 为输入，c 为输出。
qL, qR, qM, qO, qC 称为Selectors，决定门的类型。


门编号qL qR qM qO qC  a   b   c
1     0  0  1 -1  0  x   x   x^2
2     0  0  1 -1  0  y   y   y^2
3     0  0  1 -1  0  z   z   z^2
4     1  1  0 -1  0  x^2 y^2 z^2

0*x+0*x+1*x*x-1*x^2+0=0
0*y+0*y+1*y*y-1*y^2+0=0
0*z+0*z+1*z*z-1*z^2+0=0
1*x^2+1*y^2+0*x*y-1*z^2+0=0

