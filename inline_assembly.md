# 内联汇编

1. ### 基础要点

   汇编语法操作数方向

   ```assembly
   AssemblerInstructions dst src //Intel syntax
   AssemblerInstructions src dst //AT&T syntax
   ```

   

2. ### 样例分析：

   扩展内联汇编（Extended inline assembly）样例：

   ```c
   /* https://github.com/torvalds/linux/blob/d9862cfbe2099deb83f0e9c1932c91f2d9c50464/arch/x86/include/asm/string_32.h
   path: linux/arch/x86/include/asm/string_32.h 
   */
   
   static __always_inline void *__memcpy(void *to, const void *from, size_t n)
   {
   	int d0, d1, d2;
   	asm volatile("rep ; movsl\n\t"
   		     "movl %4,%%ecx\n\t"
   		     "andl $3,%%ecx\n\t"
   		     "jz 1f\n\t"
   		     "rep ; movsb\n\t"
   		     "1:"
   		     : "=&c" (d0), "=&D" (d1), "=&S" (d2)	// output operands
   		     : "0" (n / 4), "g" (n), "1" ((long)to), "2" ((long)from)	// input operands
   		     : "memory");		// clobbers
   	return to;
   }
   ```

   - `movsb`,`movsw`,`movsl`：分别传送一个字节/字/双字（1byte/2bytes/4bytes）。`movs`系列指令的源操作数与目的操作数均隐含，源操作数地址`ds:esi`，目的操作数地址`es:edi`。同时`movs`指令每执行一次，`esi`和`edi`存储的目标地址会递增或递减更新，以保证`esi`/`edi`指向下一读取/写入地址。

   - `rep`为重复前缀指令，重复次数隐含在`ecx`中，每重复执行一次`ecx`中的值会减`1`。

   - `\n\t`打印回车符号到汇编文件。
   - 输入操作符列表中约束可使用数字替代具体操作符，从首个输出操作符以`0`起编码（`%4`对应操作数`"g"(n)`）。
   - 输出操作数必须为左值，输入操作数无此限制。
   - `&`约束表明是一个受早期影响的操作数（操作数内容变化先于指令完成，比如`esi`、`edi`在`movs`执行过程中已变化）。
     - `jmp`指令可使用数值标签：`[0-9]:`。
     - 当数值标签作为跳转引用时数值标签需搭配后缀`f`（forward）/`b`（backward），如：`jmp  1f/1b`。

   代码解释：

   输入操作数表达式`n/4`与输出操作数`d0`共用寄存器`ecx`，输入操作数`to`、`from`分别与`d1`、`d2`共用寄存器`edi`、`esi`，输入    操作数`n`由编译器选择除通用寄存器之外的寄存器、内存、立即数。先执行`"rep ; movsl\n\t"`，直到`ecx`减至`0`，而后将操作数`n`读入`ecx`，并将`ecx`值与立即数`3`作位与运算，结果写入`ecx`。若`n%4 == 0`此时`ecx`值为`0`同时修改标志位`ZF`，由`jz 1f`前向跳至标签`"1:"`处；若`n%4 != 0`此时`ecx`值更新为`n%4`的余数且保持标志位`ZF`，`jz`指令不执行，由`rep; movsb\n\t`完成剩余字节的拷贝。受影响列表除去输出操作数中的寄存器都需要纳入，

   

1. ### 基本内联汇编(**Basic inline assembly**)

   **基本语法模板**

   ```assembly
   asm asm-qualifiers( AssemblerInstructions )
   ```

   **关键字：**`asm`、`__asm__`

   **修饰符（Qualifiers ）：**`volatile`、`inline`

   基本内联汇编语句块(basic asm blocks)隐式声明为`volatile`。

   > **volatile**	The optional volatile qualifier has no effect. All basic asm blocks are implicitly volatile 
   >
   > **inline**	If you use the inline qualifier, then for inlining purposes the size of the asm
   > statement is taken as the smallest size possible 
   >
   > <!--on page 572 of Using the GNU Compiler Collection (For gcc version 10.0.0 (pre-release) )-->

   **批注（Remarks）：除以下两种情况，使用扩展内联汇编**

   - C函数体外使用内联汇编
   - `naked`修饰的函数

   > Using extended asm (see Section 6.47.2 [Extended Asm], page 574) typically produces
   > smaller, safer, and more efficient code, and in most cases it is a better solution than basic asm. However, there are two situations where only basic asm can be used: 
   >
   > - Extended asm statements have to be inside a C function, so to write inline assembly language at file scope (“top-level”), outside of C functions, you must use basic asm. You can use this technique to emit assembler directives, define assembly language macros that can be invoked elsewhere in the file, or write entire functions in assembly language. Basic asm statements outside of functions may not use any qualifiers 
   > - Functions declared with the naked attribute also require basic asm (see Section 6.33 [Function Attributes], page 481). on page 572 of Using the GNU Compiler Collection (For gcc version 10.0.0 (pre-release) )
   >
   > <!--on page 573 of Using the GNU Compiler Collection (For gcc version 10.0.0 (pre-release) )-->

   

2. ### 扩展内联汇编(**Extended inline assembly**)

   **基本语法模板**

   ```assembly
   asm asm-qualifiers (AssemblyTemplate
                      :OutputOperands
                      [:InputOperands
                      [:Clobbers]])
   					
   asm asm-qualifiers (AssemblyTemplate
                      :					//empty
                      :InputOperands
                      :Clobbers
                      :GotoLabels)
   ```

   **修饰符（Qualifiers ）：**`volatile`、`inline`、`goto`

   > **volatile**	The typical use of extended asm statements is to manipulate input values to produce output values. However, your asm statements may also produce side effects. If so, you may need to use the volatile qualifier to disable certain optimizations. See [Volatile], page 575. 
   >
   > **inline**	If you use the inline qualifier, then for inlining purposes the size of the asm statement is taken as the smallest size possible (see Section 6.47.6 [Size of an asm], page 627).
   >
   > **goto**	This qualifier informs the compiler that the asm statement may perform a jump to one of the labels listed in the GotoLabels[GotoLabels], page 587.
   >
   > <!--on page 574 of Using the GNU Compiler Collection (For gcc version 10.0.0 (pre-release) )-->

   **参数列表（Parameters ）：** `AssemblyTemplate`、`OutputOperands`、`InputOperands`、`Clobbers`、`GotoLabels`

   > **AssemblyTemplate**	This is a literal string that is the template for the assembler code. It is a combination of fixed text and tokens that refer to the input, output, and goto parameters. See [AssemblerTemplate], page 577.
   >
   > **OutputOperands**	A comma-separated list of the C variables modified by the instructions in the AssemblerTemplate. An empty list is permitted. See [OutputOperands], page 579. 
   >
   > **InputOperands**	A comma-separated list of C expressions read by the instructions in the AssemblerTemplate. An empty list is permitted. See [InputOperands], page 582.
   >
   > **Clobbers**	A comma-separated list of registers or other values changed by the AssemblerTemplate, beyond those listed as outputs. An empty list is permitted. See [Clobbers and Scratch Registers], page 584.
   >
   > **GotoLabels**	When you are using the goto form of asm, this section contains the list of all C labels to which the code in the AssemblerTemplate may jump. See [GotoLabels], page 587.asm statements may not perform jumps into other asm statements, only to the listed GotoLabels. GCC’s optimizers do not know about other jumps; therefore they cannot take account of them when deciding how to optimize. 
   >
   > ***The total number of input + output + goto operands is limited to 30***
   >
   > <!--on page 574 of Using the GNU Compiler Collection (For gcc version 10.0.0 (pre-release) )-->

   #### `volatile`

   GCC在优化的过程中，被判定不需要的输出变量所在的汇编语句块会被舍弃。同时，被判定每次的输出结果都相同的汇编语句块会被GCC优化器移出循环从而舍弃。`volatile`修饰符会阻止GCC优化被修饰汇编语句块。无输出操作数(output operands)且包含`goto`的汇编语句块会隐式(implicitly)声明为`volatile`。

   样例1:arrow_down_small:

   ```c
   void read_time_stamp_counter() {
     uint64_t msr;
     __asm__ volatile(
         "rdtsc\n\t"           // return the time in EDX:EAX
         "shl $32, %%rdx\n\t"  // shift the upper bits left
         "or %%rdx, %0"        //'or' in the lower bits
         : "=a"(msr)
         :                     // empty
         : "rdx");
     printf("msr: %llx\n", msr);
   }
   ```

   - `or`指令执行逻辑或操作，结果放入第二个操作符。
   - `rdtsc`将结果写入`EDX:EAX`。
   - `shl`执行逻辑左移，结果放入第二个操作数。

   `rdtsc`将结果写入`EDX:EAX`，而后对`rdx`逻辑左移32位将`edx`值搬至`rdx`高32位，而后对`rdx`和`rax`或操作将结果写入`rax`。

   > The following example demonstrates a case where you need to use the volatile qualifier. It uses the x86 `rdtsc`instruction, which reads the computer’s time-stamp counter. Without the volatile qualifier, the optimizers might assume that the asm block will always return the same value and therefore optimize away the second call. 
   >
   > <!--on page 576 of Using the GNU Compiler Collection (For gcc version 10.0.0 (pre-release) )-->

   #### **特殊格式字符串**（Special format strings）：

   `%%`、`%{`、`%|`、`%}`分别输出`%`、`{`、`|`、`}`字符到汇编代码。

   `%=`输出一具有唯一性的数字，对于创建需要多次引用的标签非常有用。

   > **%=**	Outputs a number that is unique to each instance of the asm statement in the entire compilation. This option is useful when creating local labels and referring to them multiple times in a single template that generates multiple assembler instructions.
   >
   > <!--on page 578 of Using the GNU Compiler Collection (For gcc version 10.0.0 (pre-release) )-->

   #### 输出操作数（Output Operands）：

   ```c
   bool old;
   __asm__ ("btsl %2,%1\n\t"	//turn on zero-based bit #Offset in Base
            "sbb %0,%0"		//use the CF to calculate old
           :"=r"(old),"+rm"(*Base)		//outputs
           :"Ir"(Offset)			//inputs
           :"cc");			//clobbers
   return old;
   //	%0 => old
   //	%1 => *Base
   //	%2 => Offset
   ```

   操作数格式：`[ [asmSymbolicName] ] constraint (cVariableName)`

   <u>标签引用（asmSymbolicName）</u>：对周围变量、内联汇编操作数的进行引用，格式为`%[Value]`。

   <u>非标签引用</u>：根据操作数（输出/输入）先后顺序从零开始编号，格式为`%number`。

   <u>约束（constraint ）</u>：`=`  写变量，变量可不存在；`+`  可读写变量；`r` 寄存器；`m` 内存。`=rm` 写变量，同时指定寄存器和内存，编译器会基于当前内容选择最有效方式。

   <u>C变量名（cVariableName）</u>：输出操作数变量必须是左值，输入操作数变量无限制。

   已修改寄存器（clobbered registers）：作为输出操作数的寄存器不需写入已修改列表。

   **标签引用示例：**

   ```c
   uint32_t c = 1;
   uint32_t d;
   uint32_t *e = &c;
   asm ("mov %[e], %[d]"
       :[d]"=rm"(d)
       :[e]"rm"(*e));
   ```

   #### 输出标志操作数（Flag Output Operands）：

   存在一些目标用特殊寄存器存储操作结果的标志。一般这些寄存器不会被汇编语句修改，除非认定会篡改寄存器内容。

   > Some targets have a special register that holds the “flags” for the result of an operation or comparison. Normally, the contents of that register are either unmodifed by the asm, or the asm statement is considered to clobber the contents. 
   >
   > <!--on page 581 of Using the GNU Compiler Collection (For gcc version 10.0.0 (pre-release) )-->

   #### 输入操作数（Input Operands）：

   操作数格式：`[ [asmSymbolicName] ] constraint (cExpression)`

   输入操作数可使用标签/数字序，将输入变量与某个输出变量共用同一个寄存器/内存/其它.

   #### 受影响位置及寄存器（Clobbers and Scratch Registers ）：

   受影响列表需列出除输出操作数外的所有由汇编代码块修改的位置。

   > While the compiler is aware of changes to entries listed in the output operands, the inline asm code may modify more than just the outputs. 
   >
   > <!--on page 584 of Using the GNU Compiler Collection (For gcc version 10.0.0 (pre-release) )-->

   `cc`：	汇编代码修改了标志寄存器。

   > The "cc" clobber indicates that the assembler code modifies the flags register. 

   `memory`：	汇编代码对内存进行了读/写操作。

   > The "memory" clobber tells the compiler that the assembly code performs memory reads or writes to items other than those listed in the input and output operands (for example, accessing the memory pointed to by one of the input parameters). 

   #### 跳转标签（Goto Labels）：

   示例：

   ```c
   asm goto ("btl %1, %0\n\t"
            "jc %l2"			//%l+number or %l[laberName]
            :					//empty
            :"r"(p1),"r"(p2)
            :"cc"
            :carry);
   return 0;
   
   carry:
   return 1;
   ```

   #### x86操作数修饰符（x86 Operand Modifiers）：

   没有修饰符，以下是操作数对AT&T和Intel汇编格式的输出：

   | Operand | at&t   | intel             |
   | ------- | ------ | ----------------- |
   | `%0`    | `%eax` | `eax`             |
   | `%1`    | `$2`   | `2`               |
   | `%3`    | `$.L3` | `OFFSET FLAT:.L3` |

   再次基础上，列出增加修饰符后的汇编代码输出。

   | Modifier          | Description                                                  | Operand |   AT&T    |  Intel   |
   | ----------------- | ------------------------------------------------------------ | :-----: | :-------: | :------: |
   | `a`               | absolute memory reference                                    |  `%A0`  |  `*%rax`  |  `rax`   |
   | `b`               | QImode(四分之一整型 1byte) name of the register              |  `%b0`  |   `%al`   |   `al`   |
   | `c`(lowercase)    | print the constant expression with no punctuation            |  `%c1`  |    `2`    |   `2`    |
   | `E`               | address in Double Integer mode (双整形 8bytes) <br />when the target is 64-bit |  `%E1`  | `%(rax)`  | `[rax]`  |
   | `h`               | QImode（1byte） name for a “high” register.                  |  `%h0`  |   `%ah`   |   `ah`   |
   | `H`               | Add 8 bytes to an offsettable memory reference               |  `%H0`  | `8%(rax)` | `8[rax]` |
   | `k`(lowercase)    | SImode（4bytes） name of the register                        |  `%k0`  |  `%eax`   |  `eax`   |
   | `l`(lowercase  L) | label name                                                   |  `%l3`  |   `.L3`   |  `.L3`   |
   | `p`               | raw symbol name(without syntax specific prefixes)            |  `%p2`  |   `42`    |   `42`   |
   | `P`(uppercase)    |                                                              |         |           |          |
   | `q`               | DImode（8bytes） name of the register                        |  `%q0`  |  `%rax`   |  `rax`   |
   | `w`               | HImode（二分之一整型 2bytes） name of the register           |  `%w0`  |   `%ax`   |   `ax`   |
   | `z`               | opcode suffix for the size of the current integer<br /> operand (one of b/w/l/q) |  `%z0`  |    `l`    |          |
   | `V`(uppercase)    | a special modifier which prints the name of the<br /> full integer register without % |         |           |          |

   <!--on page 589 of Using the GNU Compiler Collection (For gcc version 10.0.0 (pre-release) )-->

3. ### 操作数约束（Constraints for asm Operands）:

   #### 简单约束（Simple Constraints）：

   | Constraints                   | Description                                                  |
   | ----------------------------- | ------------------------------------------------------------ |
   | '`whitespace`'                | 空格会被忽略，用作代码对齐<br />Whitespace characters are ignored and can be inserted at any position except the first. |
   | '`m`'                         | 允许使用内存，任意地址<br />A memory operand is allowed, with any kind of address that the machine supports in general. |
   | '`o`'                         | 允许使用地址可偏移（offsettable）的内存<br />A memory operand is allowed, but only if the address is offsettable. |
   | '`V`'（uppercase）            | 允许使用不可偏移的内存，与`o`对立<br />A memory operand that is not offsettable. In other words, anything that would fit the ‘m’ constraint but not the ‘o’ constraint. |
   | '`<`'                         | A memory operand with autodecrement addressing (either predecrement or postdecrement) is allowed. |
   | '`>`'                         | A memory operand with autoincrement addressing (either preincrement or postincrement) is allowed. |
   | '`r`'                         | 允许使用通用寄存器（`eax`,`ebx`,`ecx`,`edx`,`esi`,`edi`,`ebp`,`esp`）<br />A register operand is allowed provided that it is in a general register. |
   | '`i`'                         | 允许使用立即数整数（包含符号常量）<br />An immediate integer operand (one with constant value) is allowed. This includes symbolic constants whose values will be known only at assembly time or later. |
   | '`n`'                         | 允许使用已知数值的立即数整数<br />An immediate integer operand (one with constant value) is allowed. This includes symbolic constants whose values will be known only at assembly time or later. |
   | '`I`','`J`','`K`',`...`,'`P`' | 允许使用与机器相关且在一定范围内的立即整数操作数<br />Other letters in the range ‘I’ through ‘P’ may be defined in a machine-dependent fashion to permit immediate integer operands with explicit integer values in specified ranges. |
   | '`E`'                         | 允许使用立即浮动操作数，条件是：浮点数格式与机器匹配<br />An immediate floating operand (expression code `const_double`) is allowed, but only if the target floating point format is the same as that of the host machine (on which the compiler is running). |
   | '`F`'                         | 允许使用立即浮动操作数<br />An immediate floating operand (expression code `const_double` or `const_vector`) is allowed. |
   | '`G`','`H`'                   | 允许使用与机器相关且在一定范围内的立即浮动操作数<br />‘G’ and ‘H’ may be defined in a machine-dependent fashion to permit immediate floating operands in particular ranges of values. |
   | '`s`'（lowercase）            | 允许使用其值不是显式整数的立即整数操作数<br />An immediate integer operand whose value is not an explicit integer is allowed. Sometimes it allows better code to be generated. |
   | '`g`'                         | 允许使用寄存器、内存和立即整数操作数，不支持通用寄存器<br />Any register, memory or immediate integer operand is allowed, except for registers that are not general registers. |
   | '`X`'（uppercase）            | 允许使用任意允许的操作数<br />Any operand whatsoever is allowed. |
   | '`0`','`1`','`3`','...','`9`' | An operand that matches the specified operand number is allowed. |
   | '`p`'（lowercase）            | 允许使用具有有效内存地址的操作数<br />An operand that is a valid memory address is allowed. This is for `load_address` and `push_address` instructions. |
   | `other-letters`               | Other letters can be defined in machine-dependent fashion to stand for particular classes of registers or other arbitrary operand types. ‘d’, ‘a’ and ‘f’ are defined on the 68000/68020 to stand for data, address and floating point registers. |

   <!--on pp.590-593 of Using the GNU Compiler Collection (For gcc version 10.0.0 (pre-release) )-->

   **x86系列特定约束（x86 family）：**

   | Constraints    | Description                                                  |
   | -------------- | ------------------------------------------------------------ |
   | `R`            | Legacy register — the eight integer registers available on all i386 processors (`a`, `b`, `c`, `d`, `si`, di, `bp`, `sp`). |
   | `q`            | Any register accessible as `rl`. In 32-bit mode, `a`, `b`, `c`, and `d`; in 64-bit mode, any integer register. |
   | `Q`            | Any register accessible as `rh`: `a`, `b`, `c`, and `d`.     |
   | `a`            | `eax`    (The a register)                                    |
   | `b`            | `ebx`                                                        |
   | `c`            | `ecx`                                                        |
   | `d`            | `edx`                                                        |
   | `S`(uppercase) | `esi`   (The si register)                                    |
   | `D`            | `edi `   (The di register)                                   |
   | `A`            | The `a` and `d` registers. This class is used for instructions that re turn double word results in the `ax:dx` register pair. |
   | `U`            | The call-clobbered integer registers.                        |
   | `f`            | Any 80387 floating-point (stack) register.                   |
   | `t`            | Top of 80387 floating-point stack (`%st(0)`).                |
   | `u`(lowercase) | Second from top of 80387 floating-point stack (`%st(1)`).    |
   | `y`            | Any MMX register                                             |
   | `x`(lowercase) | Any SSE register                                             |
   | `v`(lowercase) | Any EVEX encodable SSE register (`%xmm0-%xmm31`).            |
   | `Yz`           | First SSE register (`%xmm0`).                                |
   | `I`            | Integer constant in the range `0 ... 31`, for 32-bit shifts. |
   | `J`            | Integer constant in the range `0 ... 63`, for 64-bit shifts. |
   | `K`(uppercase) | Signed 8-bit integer constant.                               |
   | `L`            | `0xFF` or `0xFFFF`, for andsi as a zero-extending move.      |
   | `M`            | 0, 1, 2, or 3 (shifts for the `lea` instruction).            |
   | `N`            | Unsigned 8-bit integer constant (for `in` and `out` instructions). |
   | `G`            | Standard 80387 floating point constant.                      |
   | `C`(uppercase) | SSE constant zero operand.                                   |
   | `e`            | 32-bit signed integer constant, or a symbolic reference known to fit that range (for immediate operands in sign-extending x86-64 instructions). |
   | `We`           | 32-bit signed integer constant, or a symbolic reference known to fit that range (for sign-extending conversion operations that require non-VOIDmode immediate operands). |
   | `Wz`           | 32-bit unsigned integer constant, or a symbolic reference known to fit that range (for zero-extending conversion operations that require non-VOIDmode immediate operands). |

   <!--on page 621 of Using the GNU Compiler Collection (For gcc version 10.0.0 (pre-release) )-->

   **IA-64系列约束：**

   | Constraints    | Description                                                  |
   | -------------- | ------------------------------------------------------------ |
   | `a`            | General register `r0` to `r3` for `addl` instruction         |
   | `b`            | Branch register                                              |
   | `c`            | Predicate register (‘`c`’ as in “conditional”)               |
   | `d`            | Application register residing in M-unit                      |
   | `e`            | Application register residing in I-unit                      |
   | `f`            | Floating-point register                                      |
   | `m`            | Memory operand. If used together with ‘`<`’ or ‘`>`’, the operand can have postincrement and postdecrement which require printing with ‘`%Pn`’ on IA-64. |
   | `G`            | Floating-point constant `0.0` or `1.0`                       |
   | `I`            | 14-bit signed integer constant                               |
   | `J`            | 22-bit signed integer constant                               |
   | `K`(uppercase) | 8-bit signed integer constant for logical instructions       |
   | `L`            | 8-bit adjusted signed integer constant for compare pseudo-ops |
   | `M`            | 6-bit unsigned integer constant for shift counts             |
   | `N`            | 9-bit signed integer constant for load and store postincrements |
   | `O`(uppercase) | The constant zero                                            |
   | `P`            | 0 or -1 for `dep` instruction                                |
   | `Q`            | Non-volatile memory for floating-point loads and stores      |
   | `R`            | Integer constant in the range `1` to `4` for `shladd` instruction |
   | `S`(uppercase) | Memory operand except postincrement and postdecrement. This is now roughly the same as ‘`m`’ when not used together with ‘`<`’ or ‘`>`’ |

   <!--on page 603 of Using the GNU Compiler Collection (For gcc version 10.0.0 (pre-release) )-->

   #### 多可选约束（Multiple Alternative Constraint）：

   ```assembly
   //alternative 0
   asm ("or %0, %1"
   	:"+m"(output)
   	:"ir"(input)
   	:<i dont konw>)
   	
   //alternative 1
   asm ("or %0, %1"
   	:"+r"(output)
   	:"irm"(input)
   	:<i dont konw>)
   	
   //multiple alternative constraint
   asm ("or %0, %1"
   	:"+m,r"(output)
   	:"ir,irm"(input)
   	:<i dont konw>)
   ```

   > Sometimes a single instruction has multiple alternative sets of possible operands. For example, on the 68000, a logical-or instruction can combine register or an immediate value into memory, or it can combine any kind of operand into a register; but it cannot combine one memory location into another.
   >
   > These constraints are represented as multiple alternatives. An alternative can be described by a series of letters for each operand. The overall constraint for an operand is made from the letters for this operand from the first alternative, a comma, the letters for this operand from the second alternative, a comma, and so on until the last alternative. All operands for a single instruction must have the same number of alternatives.
   >
   > So the first alternative for the 68000’s logical-or could be written as `"+m" (output): "ir" (input)`. The second could be `"+r" (output): "irm" (input)`. However, the fact that two memory locations cannot be used in a single instruction prevents simply using `"+rm" (output) : "irm" (input)`. Using multi-alternatives, this might be written as `"+m,r (output) : "ir,irm" (input)`. This describes all the available alternatives to the compiler, allowing it to choose the most efficient one for the current conditions.
   >
   > There is no way within the template to determine which alternative was chosen. However you may be able to wrap your asm statements with builtins such as `__builtin_constant_p` to achieve the desired results.
   >
   > <!--on page 593 of Using the GNU Compiler Collection (For gcc version 10.0.0 (pre-release) )-->

   #### 约束修饰符（Constraint Modifier Characters ）：

   | Character | Descriptions                                                 |
   | --------- | ------------------------------------------------------------ |
   | `=`       | 操作符写操作<br />Means that this operand is written to by this instruction: the previous value is discarded and replaced by new data. |
   | `+`       | 操作符读写操作<br />Means that this operand is both read and written by the instruction. |
   | `&`       | 操作符内容变化先于指令完成<br />Means (in a particular alternative) that this operand is an earlyclobber operand, which is written before the instruction is finished using the input operands. |
   | `%`       | Declares the instruction to be commutative for this operand and the following operand. This means that the compiler may interchange the two operands if that is the cheapest way to make all operands fit the constraints. ‘%’ applies to all alternatives and must appear as the first character in the constraint. Only read-only operands can use ‘%’. |

   
