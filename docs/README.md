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

[EVM Circuit]

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

