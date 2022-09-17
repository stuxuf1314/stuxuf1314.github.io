# JIT编译过程

JIT(Just-in-Time)即时编译指的是程序运行期间执行编译的一种方式：从一种二进制格式到另外一种二进制格式的翻译过程。一般是从字节码（bytecode）到机器码（machine code），例如java/OCaml/eBPF字节码到arm32二进制指令。

整个翻译过程包括源和目标的二进制格式，对应的编码解码以及核心的翻译规则。

# 编码和解码
- `decode: 'A -> option 'syntax`, 从01二进制格式转换为可读格式（例如归纳定义的ISA syntax）,解码过程可能会失败（如果二进制格式有问题）
- `encode: syntax -> 'A`,反过来一般不会失败

通过`decode`函数，我们可以基于已有的形式化定义（例如CompCert支持的ISAs）来讨论JIT编译过程的正确性。


我们考虑一个eBPF字节码到arm32的例子：div32。

# eBPF字节码
eBPF ISA
- 每个指令都是64 bit
- 寄存器部分 R0-R10 registers + PC

## eBPF编码格式
```shell
6 6 6 5 5 5 5 5 4 4 4 4 4 3 3 3 3 3 2 2 2 2 2 1 1 1 1 1 0 0 0 0 0
4 2 0 8 6 4 2 0 8 6 4 2 0 8 6 4 2 0 8 6 4 2 0 8 6 4 2 0 8 6 4 2 0
 ----------------------------------------------------------------
 |           imm32              |     offset    |src|dst|opcode |
 ----------------------------------------------------------------
```
- 操作码opcode = 8 bit
- dst (src) = 4 bit
- 偏移量offset = 16 bit （跳转指令会涉及）
- 立即数imm = 32 bit

## eBPF `div32 R0 R1`
eBPF div32的操作码为0x`3c`
- `decode_eBPF(0x103c) = Some (div32 R0 R1)`
- `encode_eBPF(div32 R0 R1) = 0x103c`

# Arm32字节码

ARM32 ISA
- 每个指令都是32 bit
- 寄存器部分 R0-R12 + SP (stack pointer) + LR (link register) + PC

## ARM32-v7-A-A1 编码格式
```shell
- UDIV Rd Rn Rm := [|Rd|]  <- [|Rn|] / [|Rm|]

0x0730f010 = 0b 01110 011 0000 1111 0000 0001 0000

 3 3 2 2 2 2 2 2 2 2 2 2 1 1 1 1 1 1 1 1 1 1 0 0 0 0 0 0 0 0 0 0
 1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0
----------------------------------------------------------------
| Cond  |0 1 1 1 0|0 1 1|   Rd  |1 1 1 1|  Rm   |0 0 0|1| Rn   |
----------------------------------------------------------------
```

## arm32 `udiv R0 R1`
- `decode_arm32(0x0730f110) = Some (div R0 R1)`
- `encode_arm32(div R0 R1) = 0x0730f110`


## JIT 翻译规则
- `jit_div32_rule`: eBPF `div32 dst src` -> arm32 `udiv dst dst src`

所以这里完整的JIT编译(`jit_eBPF_to_arm32: list int64 -> list int`)中针对`div32`部分 （`jit_eBPF_to_arm32_div32: int64 -> int`）= `decode_eBPF + jit_div32_rule + encode_arm32`

