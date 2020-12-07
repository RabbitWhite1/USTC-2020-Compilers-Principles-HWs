# 中间代码与汇编码理解

## step1

### 1. 分析 src/data.c 中的变量在不同形式下的不同表示与访问方式

#### x86-64

##### aa 和 bb

对于全局变量来说, 就直接在 `.data` 下声明了:
- `aa`: 
    ```asm
        .align 8                # 对齐
        .type	aa, @object     # 类型
        .size	aa, 8           # 大小 8 bytes
    aa:
        .quad	10              # 初始化值 10
    ```
- `bb`:
    ```asm
        .globl	bb              # 全局
        .align 2                # 对齐
        .type	bb, @object     # 类型
        .size	bb, 2           # 大小 2 bytes
    bb:
        .value	20              # 初始化值 20
    ```

##### cc

对于函数内的静态变量, 则被存在了静态区, 且赋予了初始化值:
```asm
	.align 8                    # 对齐
	.type	cc.1796, @object    # 类型
	.size	cc.1796, 8          # 大小 8 bytes
cc.1796:
	.quad	30                  # 双字初始化为 30
```
值得注意的是, 这里的 `cc` 被加上了后缀: `cc.1796`, 这可以用以区分其他函数内的同名变量.

##### dd

对于函数内非静态变量, 则存在于 `栈` 中, 注意其中 `%rbq` 是基址寄存器, `%rsp` 是栈指针寄存器. 从下面这句可以看出来, `dd` 被存在了 `%rbp - 2` 的地方.
```asm
movw	$40, -2(%rbp)
```

#### x86-32

它与 `x86-64` 主要区别如下:
- 使用 `long` 而不是 `quad`: 其 `long` 类型是 4 bytes 的(如`.size aa, 4`)
- 使用 `%ebp` 而不是 `%rbp`

其他的基本和 `x86-64` 一致, 故不多描述了.

#### clang 64 和 32 位

都是些类似的差别, 比如
- 用 `short` 而非 `value`
- 一个重要区别: 用 `func.cc` 而不是 `cc.1796` 之类的, 这和 `llvm` 能够对应起来

#### arm

- 对于 `aa`, 其和 `x86-64` 主要区别是:
    - 用了 `.xword` 而非 `quad`
- 对于 `bb`, 其和 `x86-64` 主要区别是:
    - 用了 `hword` 而非 `value`
- 对于 `aa`, 其和 `x86-64` 主要区别是:
    - 用了 `.xword` 而非 `quad`
- 对于 `dd`, 其和 `x86-64` 主要区别是:
    - 它位于 `sp + 14` 的位置. 这里 `sp` 是 `arm` 里的 `栈指针寄存器`

#### LLVM IR

除了根据机器位数不同作了一些调整, 根据一些版本信息有所不同以外, `LLVM IR` 的核心部分代码都是一样的. 以 `x86-64` 对应的为例:

- `aa`: 未被定义
- `bb`:
    ```llvm
    @bb = dso_local global i16 20, align 2
    ```
    - `dso_local`: 提示编译器这个变量将在同一个 `链接单元` 被解析为一个符号.
    - `global`: 表示其为全局变量
    - `i16`: 表示了类型
    - `20`: 表示了初值
    - `align 2`: 表示了对齐
- `cc`:
    ```llvm
    @func.cc = internal global i64 30, align 8
    ```
    - `internal`: 表示 `链接类型`, 在目标文件中是一个本地符号.
    - 其他的类似
- `dd`: llvm 用 `alloca` 为其分配了空间, 并 `store` 了 `40` 进那个空间.
    ```llvm
    %1 = alloca i16, align 2
    store i16 40, i16* %1, align 2
    ```

### 2. `step1/data1.c` 及其分析

#### x86

显然, `x86-64` 模式下, 不论是 `clang` 还是 `gcc`, 其结果步骤都差不多, 即
1. 声明对应常量
2. 通过 `mov` 操作移动常量
3. 通过类型转换指令做类型转换
4. 做加法
5. 存数

而在 `x86 32位` 模式下, `clang` 和 `gcc` 结果也基本上一致. 但由于没有足够大的寄存器可以使用, 故对于 `double` 类型变量有一些比较复杂的操作.

#### arm

在 `64位的 arm` 之下, 其结果与 `x86-64` 其实也差不多, 区别基本上还是前面提到的用的是 `sp` 等东西, 指令略有不同等(如用 `fcvtzs` 进行类型转换)