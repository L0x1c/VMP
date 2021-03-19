## VMP

### 搭建环境

主要的模拟环境是capstone和unicorn

capstone: https://github.com/aquynh/capstone

unicorn: https://github.com/unicorn-engine/unicorn

VMProtect 3.5.0: https://down.52pojie.cn/Tools/Packers/

测试环境代码: (很重要！)：https://github.com/L0x1c/VMP/tree/main/vmp%20notes/test



正常测试结果：

![image-20210104102840594](VMP.assets/image-20210104102840594.png)



### 模拟环境

最开始的时候我们将优化关掉，之后黑盒子测试的时候会将优化打开看看加了vmp都有什么区别的

![image-20210104103131177](VMP.assets/image-20210104103131177.png)

样本代码：

```c
#include <stdio.h>
#include <stdlib.h>
#include <Windows.h>
#pragma warning(disable : 4996)

char buf[1204];

void main()
{
	while (1)
	{
		scanf("%s", buf);
		if (!strcmp(buf, "123"))
			printf("ok\n");
		else
			printf("fail\n");c
	}

}
```

如果我们开启优化的话我们可以看到像strcmp这样的函数将会内嵌到main中：

![image-20210104103632926](VMP.assets/image-20210104103632926.png)

但是如果我们不启用优化 就可以看到是进行call然后外平栈很传统的一个方式：

![image-20210104103709176](VMP.assets/image-20210104103709176.png)

我们将样本加vmp壳：

![image-20210104103944274](VMP.assets/image-20210104103944274.png)

我们拖到xdbg进行调试，定位到main以后看一下

![image-20210104104412656](VMP.assets/image-20210104104412656.png)

这里的push call相当于一个进入虚拟机的一个标志，相当于假call，我们在call下面进行断点的时候会发现，无法断下来，因为已经跑飞了

![image-20210104104430303](VMP.assets/image-20210104104430303.png)

测试就用CE来测试我们这个是假的call，可以看到我们程序跑起来了后我们这里没有断下来切这个位置是cc

![image-20210104105007834](VMP.assets/image-20210104105007834.png)

我们现在需要模拟环境，所以用模拟的环境来跑我们的vmp

因为我们当前的EIP在.vmp0的节中，还有我们当前的线程堆栈的状态，我们需要dump下来，一个大小是0x8A000一个是5000

![image-20210104105248535](VMP.assets/image-20210104105248535.png)

需要修改我们的当前的寄存器的状态在我们的模拟环境中，和当前dump下来内存的大小和名字地址

![image-20210104105702230](VMP.assets/image-20210104105702230.png)

因为我们的模拟器是while(1)的，所以我们跑的时候一定会出现一定的异常比如说我们的scanf需要进内核但是我们没有内存映射所以就会产生异常而中断下来，这里面模拟器的一些参数比较有用：

这里可以看到我们模拟成功了，和我们这边的参数是一样的，address: 0x485b76相对应

![image-20210104110109592](VMP.assets/image-20210104110109592.png)

比较有用的就是我们这里的detail

detail->x86->op_count ： 表示我们有多少个操作数

operands：

|      | 名称         | 值                                                           | 类型      |
| ---- | ------------ | ------------------------------------------------------------ | --------- |
| ▶    | [0x00000000] | {type=X86_OP_IMM (0x00000002) reg=0x4cc49084 imm=0x000000004cc49084 ...} | cs_x86_op |

![image-20210104110552199](VMP.assets/image-20210104110552199.png)

detail中的regs_read这几个，代表了隐式读，隐式写

隐式读，隐式写：举个例子，push ebp我们看到这个指令都知道我们要对ebp进行压栈的操作，所以对ebp有个读的操作，这个操作叫做显示读，但是我们也会对esp进行读写的操作，所以对于esp有个隐式读隐式写的操作，那么这个操作的作用就是有什么我们会对eflag也就是标志寄存器进行隐式读写的操作，这个到后面进行污点分析等等的有所用处

![image-20210104110809509](VMP.assets/image-20210104110809509.png)

我们现在让他跑起来，可以看到这里报了一个读到了一个没有映射地址的地方我们过去看一下0x473594

![image-20210104111611563](VMP.assets/image-20210104111611563.png)

这里稍微提一下eflags的TF位，我们的TF位是的变动和我们的断点的机制有关这个xdbg我们下断点跑起来的时候我们的TF位会变成1如果我们单步的话就不会有这个问题的 (可以自己稍微尝试一下)，这里可以看到我们的403370是我们输入的buffer，4020108是我们的scanf的参数%s，47295d是我们的返回地址，4010c0是我们scanf的地址

![image-20210104112157541](VMP.assets/image-20210104112157541.png)

我们既然知道这些，我们就可以模拟scanf，因为像系统的一些dll等等的系统库，不可能vmp将所有的都加上了vmp所以我们当调用他们的时候是没有vmp的且模拟器会中断，需要自己模拟类似于scanf的这些函数去进行继续执行

模拟scanf代码：我们当前的eip在0x473594我们断下来，让我们的buffer中的值等于我们想要的值，修改我们的eip是我们的返回地址，esp当前+8，修改到我们调用完scanf的地址即可，因为我们用到了data和rdata的位置我们需要dump下来

```c
case 0x473594:
{
	DWORD val = 0x00343332;
	uc_mem_write(uc, 0x403370, &val, 4);
	regs.regs.r_eip = 0x47295d;
	uc_reg_write(uc, UC_X86_REG_EIP, &regs.regs.r_eip);
	regs.regs.r_esp += 8;
	uc_reg_write(uc, UC_X86_REG_ESP, &regs.regs.r_esp);
}
```

我们继续跑 可以看到断下来的位置是我们代码中的strcmp：

![image-20210104113457598](VMP.assets/image-20210104113457598.png)

这里有个小知识点，就是我们经过call等东西，eax，ecx，edx是易变寄存器，他们经过call的时候是可以改变的，很大几率而其他的寄存器一般在发生call之后的时候不会改变，叫不易改变寄存器

模拟strcmp代码：

```c
case 0x472ecc:
{
	regs.regs.r_eax = 1;
	err = uc_reg_write(uc, X86_REG_EAX, &regs.regs.r_eax);
	regs.regs.r_eip = 0x4854c1;
	err = uc_reg_write(uc, X86_REG_EIP, &regs.regs.r_eip);
	regs.regs.r_esp += 8;
	err = uc_reg_write(uc, X86_REG_ESP, &regs.regs.r_esp);
}
```

跑起来后：

![image-20210104114253076](VMP.assets/image-20210104114253076.png)

当然我们也可以模拟代码中修改eax的值变成0看看结果：

![image-20210104114403141](VMP.assets/image-20210104114403141.png)

### 代码块

我们指令进行单步执行的时候，可以知道我们的jmp不会打扰我们的执行跳转  ( jmp 立即数 ) 可以把这个指令当作！啥也不是！没错直接给他干掉 其他的像ret啦，ja啦，je啦，jmp寄存器啦，call啦都是不一定的，所以我们将他们打印出来

```c
!strcmp(insn->mnemonic,"ret")||(!strcmp(insn->mnemonic,"jmp") && insn->detail->x86.operands[0].type != X86_OP_IMM)||!strcmp(insn->mnemonic,"call")||(insn ->mnemonic[0] == 'j' && insn->detaicl->regs_read_count == 1 && insn->detail->regs_read[0] == X86_REG_EFLAGS)
```

![image-20210104161129077](VMP.assets/image-20210104161129077.png)

这里面的ret都相当于 

```
push reg 
ret
```

因为新版本的符合一个叫做寄存器轮转的问题所以他可能jmp ebp，jmp edi等等的

![image-20210104161201580](VMP.assets/image-20210104161201580.png)

这里的ja 都是一个地址，这个是因为他会判断自己的VM_STACK 虚拟栈，栈式虚拟机实现起来方便，膨胀倍数高，是虚拟机保护的首，虚拟栈就是临时进行数据交换，VMProtect 的 EBP 寄存器就是虚拟栈的栈顶指针，如果感觉膨胀到一定的时候需要提升栈的空间防止虚拟机空间溢出，所以会有很多ja的指令去判断，是不是到了规定的大小的值

### 局部混淆

我们如果分析虚拟机的时候需要去处理一下局部混淆的问题，我们看一下我们call进去的虚拟机的样子指令，可以看到有一堆莫名其妙的指令，那么这些指令大部分都是我们的混淆指令：

![image-20210104114758588](VMP.assets/image-20210104114758588.png)

像一个虚拟机中用特别多的jmp指令，我们就可以把这些jmp的指令当作不存在，因为jmp执行的时候仅仅是跳转而且这些jmp imm的时候我们可以明确知道他跳转到了哪里，分析vmp你可以看到我们的jmp几乎百分之99都是这种类型的jmp imm所以我们把这些jmp当作不存在，将跳转过去的代码和当权的代码块当作同一个代码块进行分析，需要ret call jcc等等的这些指令的时候我们当作是代码块的结束的位置

![image-20210104114939813](VMP.assets/image-20210104114939813.png)

因为我们现在是先分析局部混淆所以我们就先对这个局部混淆的位置进行分析，一直到0x40bc54，我们把这个代码都打出来从头：

![image-20210104115240781](VMP.assets/image-20210104115240781.png)

![image-20210104115539724](VMP.assets/image-20210104115539724.png)

```asm
Emulate i386 code
00485B76        push 0x4cc49084
00485B7B        call 0x441d33
00441D33        pushfd
00441D34        stc
00441D35        clc
00441D36        push edi
00441D37        rcl edi, 0x6b
00441D3A        xchg edi, edi
00441D3C        push edx
00441D3D        btr edi, edi
00441D40        push ecx
00441D41        btr di, ax
00441D45        rol edi, 0xc
00441D48        bswap cx
00441D4B        push eax
00441D4C        push esi
00441D4D        push ebx
00441D4E        clc
00441D4F        push ebp
00441D50        bswap si
00441D53        bts eax, ebx
00441D56        mov ecx, 0
00441D5B        cbw
00441D5D        push ecx
00441D5E        mov bl, 0x99
00441D60        clc
00441D61        mov esi, dword ptr [esp + 0x28]
00441D65        btr edi, edx
00441D68        not bp
00441D6B        bts edi, eax
00441D6E        ror esi, 1
00441D70        movzx eax, bp
00441D73        bsf eax, esi
00441D76        lea esi, [esi - 0x1394580a]
00441D7C        bswap esi
00441D7E        sal al, 0xe6
00441D81        sbb bx, bp
00441D84        xor esi, 0x5bf674ed
00441D8A        movsx ebp, cx
00441D8D        btc ebx, ebx
00441D90        not esi
00441D92        adc bp, 0x66ec
00441D97        stc
00441D98        bswap esi
00441D9A        and ebp, 0xa934b31
00441DA0        stc
00441DA1        lea esi, [esi + ecx]
00441DA4        lahf
00441DA5        mov ebp, esp
00441DA7        lea esp, [esp - 0xc0]
00441DAE        rcl ebx, cl
00441DB0        mov edi, ecx
00441DB2        mov ebx, esi
00441DB4        xadd di, cx
00441DB8        mov eax, 0
00441DBD        xor edi, eax
00441DBF        sub ebx, eax
00441DC1        or edi, 0x1a7d74b8
00441DC7        rcl di, 0xe5
00441DCB        lea edi, [0x441dcb]
00441DD1        rcr ecx, cl
00441DD3        mov ecx, dword ptr [esi]
00441DD5        stc
00441DD6        add esi, 4
00441DDC        xor ecx, ebx
00441DDE        jmp 0x42fdc3
0042FDC3        bswap ecx
0042FDC5        jmp 0x47eef0
0047EEF0        dec ecx
0047EEF1        stc
0047EEF2        neg ecx
0047EEF4        jmp 0x40bc45
0040BC45        add ecx, 0x2941083
0040BC4B        clc
0040BC4C        test dl, 0x4c
0040BC4F        xor ebx, ecx
0040BC51        add edi, ecx
0040BC53        push edi
0040BC54        ret
```

这里的局部混淆有个特点的总结，就是我们将所有的jmp都去掉，我们对寄存器的写操作，第二次写会覆盖第一次写的操作，所以第一次的写的命令就是混淆，这里有个小技巧就是我们假设要看ecx寄存器我们就可以在xdbg用按h点击ecx即可：

这里可以看到我们的ecx已经被ds:[esi]内存地址进行赋值了，我们的rcr的命令就无效了没有用了就相当于混淆指令可以去掉，但是上面的对ecx进行读取的操作所有不知道是不是混淆需要继续去分析的

![image-20210104120018697](VMP.assets/image-20210104120018697.png)

手动大概去除混淆的代码：

```asm
00441D33        pushfd
00441D36        push edi
00441D3C        push edx
00441D40        push ecx
00441D4B        push eax
00441D4C        push esi
00441D4D        push ebx
00441D4F        push ebp
00441D56        mov ecx, 0
00441D5D        push ecx
00441D5E        mov bl, 0x99
00441D61        mov esi, dword ptr [esp + 0x28]
00441D6E        ror esi, 1
00441D76        lea esi, [esi - 0x1394580a]
00441D7C        bswap esi
00441D84        xor esi, 0x5bf674ed
00441D90        not esi
00441D98        bswap esi
00441DA1        lea esi, [esi + ecx]
00441DA5        mov ebp, esp
00441DA7        lea esp, [esp - 0xc0]
00441DB2        mov ebx, esi
00441DB8        mov eax, 0
00441DBF        sub ebx, eax
00441DCB        lea edi, [0x441dcb]
00441DD3        mov ecx, dword ptr [esi]
00441DD6        add esi, 4
00441DDC        xor ecx, ebx
0042FDC3        bswap ecx
0047EEF0        dec ecx
0047EEF2        neg ecx
0040BC45        add ecx, 0x2941083
0040BC4C        test dl, 0x4c
0040BC4F        xor ebx, ecx
0040BC51        add edi, ecx
0040BC53        push edi
0040BC54        ret
```

我们生成字节码去测试一下生成一个新的内存空间

```
9c 57 52 51 50 56 53 55 b9 00 00 00 00 51 b3 99 8b 74 24 28 d1 ce 8d b6 f6 a7 6b ec 0f ce 81 f6 ed 74 f6 5b f7 d6 0f ce 8d 34 0e 8b ec 8d a4 24 40 ff ff ff 8b de b8 00 00 00 00 2b d8 3e 8d 3d cb 1d 44 00 8b 0e 83 c6 04 33 cb 0f c9 49 f7 d9 81 c1 83 10 94 02 f6 c2 4c 33 d9 03 f9 57 c3
```

我们先分配一个页的内存

![image-20210104143954062](VMP.assets/image-20210104143954062.png)

用ce去获取字节码赋值到内存区域修改call再跑起来

![image-20210104144748829](VMP.assets/image-20210104144748829.png)

直接f9看是否成功满足我们说的原理，可以看到是对的，如果还想继续很细致的恢复去混淆的话，就需要自己调试的时候细致去分析了

![image-20210104144834146](VMP.assets/image-20210104144834146.png)

但是如果想自动化去除混淆的话，需要细分寄存器的读写和eflags的读写规则（lea指令是不访问内存的）

- 指令的功能在于写
- 对于同一个位置的连续两次写，第一次写是无效的

（[mem]操作数中有对base和index的隐式读）

假设例子：

由于本人excel不知道怎么了，开启了发飙模式，用不了，直接用简陋的语言叙述来表示了

- 正常我们的指令，进行操作的时候，可能会对 寄存器的低8位 16-8位 32 - 16位进行分组，像一些eflags寄存器也需要分组，因为如果我们不分组细致分ZF，OF，AF这些位的时候，我们如果有两个指令对eflags进行写的操作，但是其中最重要的一个eflags的位的地方被第二次抹去了，就会导致最后的程序的错误
- 规则就是如果我们对一个地址，寄存器，eflags进行读写操作的时候，我们把他们标识成rw / r / w 的标识，正常的如果假设以ecx位序列的一列，总结出的标识序列应该是rwrwrwrwrwrwrw....这种形式 假设如果出现 wwr 我们就要把第一个w去掉，如果所有寄存器的一行都是r那么该指令无效是混淆指令，我们可以通过这样的方式去去除相应的混淆



//未完成的计划：

**（代码还没有实现，但是根据capstone应该可以实现出很好的去除混淆代码的方式）**



### 函数调用界面

这封图很好的解释了函数调用界面的类型的方式（只针对与vmp来说）

![image-20210104153817431](VMP.assets/image-20210104153817431.png)

- 第一个主要的就是我们的为进行优化版本的函数调用界面，因为他们系统用的函数需要走进内核以及走出，所以已经会进行出虚拟机，进虚拟机的操作
- 第二个函数调用界面，可以理解成我们自己写的函数比如就是图上的mystrcmp函数他进行了开启了优化，相当于将自定义函数进行了内联，加上vmp就是相当于第二张图的形式
- 第三张图函数调用界面一写debug版的编译会出现这个状况，我们会有出虚拟机，再进虚拟机，再出再进的一个标识在
- 第四张图就是如果不开优化，我们的正常的加vmp的一个形式，将我们的main和我们自定义的函数都加上vmp

测试代码：

```c
#include <stdio.h>
#include <stdlib.h>
#include <Windows.h>
#pragma warning(disable : 4996)

char buf[1204];


bool mystrcmp(char* p1, const char* p2)
{
	while (*p1 & *p2)
	{
		if (*p1 != *p2)
			return false;
		p1++;
		p2++;
	}
	if (*p1 || *p2)
		return false;
	return true;
}

void main()
{
	while (1)
	{
		scanf("%s", buf);
		if (mystrcmp(buf, "123"))
			printf("ok\n");
		else
			printf("fail\n");
	}

}
```

![image-20210104155525492](VMP.assets/image-20210104155525492.png)

因为我们分析可以发现进入虚拟机的标识就是push call 的标识 我们可以用这个call来定位分析我们的进入虚拟机的位置

![image-20210104191503983](VMP.assets/image-20210104191503983.png)

00401068这个地址是我们的printf的系统的那边的函数：

![image-20210104191833407](VMP.assets/image-20210104191833407.png)

那么主要的流程就是：

```asm
00449668        call 0x46af7e	;进入虚拟机
0043FC17        call 0x46af7e	;scanf后进入虚拟机
00467D9E        call 0x47608d	;mystrcmp后进入虚拟机
```

还会有个比较有意思的事情就是：

正常我们的cmp函数入口的位置是0x401100，但是我们f9他永远都不运行，因为入口的位置也是虚拟化中的一部分

![image-20210104192841746](VMP.assets/image-20210104192841746.png)

我们也可以通过ebp来找我们想要找的进入虚拟机的位置，假设我们现在再scanf之后的位置看一下ebp：

![image-20210104193815378](VMP.assets/image-20210104193815378.png)

scanf的进入

![image-20210104193855548](VMP.assets/image-20210104193855548.png)

进入虚拟机

![image-20210104193953757](VMP.assets/image-20210104193953757.png)

进入虚拟机

![image-20210104194151942](VMP.assets/image-20210104194151942.png)

![image-20210104194211563](VMP.assets/image-20210104194211563.png)

再出虚拟机

![image-20210104194255237](VMP.assets/image-20210104194255237.png)

满足了上面流程图的特点

### 黑盒测试

我们首先主要dump，自定义比较函数的cmp的内存的位置到printf，因为在正常的比较函数中会有我们相应的一个jcc的一个跳转的形式，我们通过黑盒测试去模拟出一个相应的指令的方式，这是原始代码的一个形式

![image-20210108120955619](VMP/image-20210108120955619.png)

主要的模拟就是我们的这个je加了vmp是什么样子

中途自己测试的时候发现一个好玩的事情，我们的模拟器的TF位置是因为我们下断点之后就会断下来，所以TF位置会置1，但是我们的正常的执行流程的下来TF的位置没有变成1，如果我们把eflags加上了TF位置的值，我们模拟器会自动认为，该句有断点所以只执行当前的一句代码，就不会向下执行了

因为是bool类型只跟我们al有关，所以将al置1的时候就是fail，如果置0的时候就是ok，所以我们可以确定当前的jcc的跳转的格式在我们的虚拟机的代码之中

![image-20210108142152715](VMP/image-20210108142152715.png)

如果我们不确定jcc的类型在虚拟机里都有什么跑的时候，我们打印一下看一下，可以发现大部分都是ja做虚拟机栈看是否溢出的一个保护措施

![image-20210108142752913](VMP/image-20210108142752913.png)

所以我们就要去想虚拟机中是怎么去实现je这一个效果的方法，自己模拟一下代码来实现一下：

原理（使用pushfd的情况）：

![image-20210108150126930](VMP/image-20210108150126930.png)

```c
#include <stdio.h>
#include <stdlib.h>
#include <Windows.h>


typedef void (*CALL)();

void p1()
{
	printf("in p1\n");
}

void p2()
{
	printf("in p2\n");
}
#define IS_ZERO(x) 1-((x)|(-x))>>31
void nojcc(int a, int b, CALL f1, CALL f2)
{
	__asm
	{
		mov eax,a				//输入的参数1 到eax
		sub eax,b				//看是否是等于0

		pushfd					//减法会打扰eflag的值
		pop eax					//把eflag的值传给eax

		and eax,0x40			//看第6位是否等于1 即ZF是否等于1
		shr eax,6				//左移6位看是否变成了1
		mov ecx,1				//ecx置1
		sub ecx,eax				//将ecx和eax相减，如果eax等于0的情况就是不相等，如果等于1的情况就是相等

		neg eax					//eax如果是1 取反等于0xffffffff 如果不是就是0
		neg ecx					//同上

		and eax,f1				//eax和f1进行相与，如果他相等了就是1那么neg就是0xffffffff 结果就是f1
		and ecx,f2				//同上

		add eax,ecx				//因为有一个是0 所以一定是有值的
		call eax				//call最后的结果即可
	}
}
void main()
{
	nojcc(1, 1, p1, p2);
	system("pause");
}
```

不使用pushfd的情况：

```c
#include <stdio.h>
#include <stdlib.h>
#include <Windows.h>

typedef void (*CALL)();

void p1()
{
	printf("in p1\n");
}

void p2()
{
	printf("in p2\n");
}

#define IS_ZERO(x) 1-((x)|(-x))>>31

void nojcc(int a, int b, CALL f1, CALL f2)
{
	__asm
	{
		mov eax,a			//eax = a 
		sub eax,b			//eax - b 看是否等于0

		mov ecx,eax			//ecx = eax
		neg eax				//eax 取反
		or eax,ecx			//如果进行抑或不是0的话他们就会等于-1
		shr eax,31			//eax留下符号位，因为-1的符号位是1，所以只有a=b的时候shr eax  31  = 0 

		mov ecx,1			//ecx = 1
		sub ecx,eax         //ecx = ecx - eax 
		mov eax,ecx			//eax = ecx

		mov ecx,1			//ecx = 1
		sub ecx,eax			//ecx = ecx - eax

		neg eax				//同上了
		neg ecx

		and eax,f1
		and ecx,f2

		add eax,ecx
		call eax
	}
}
void main()
{
	nojcc(1, 1, p1, p2);
	system("pause");
}
```

但是我们的vmp没有这么极限，他还是使用了pushfd的操作的，但是在vmp里是没有减法的，这个eflag的标志位的改变是比较复杂的

```
推理: a-b
NOT(NOT(a)+b) = a-b
NOT(a) = -a -1 (视为有符号)
NOT(-a-1+b) = a + 1 - b - 1 = a - b
```

结果标志位: Z , S...

过程标志位: C , O...

虽然说假设我们知道结果可以判断一个Z或者S位的东西，但是我们的过程标志是有可能发成改变的，假设0+1 = 0xFFFFFFFF + 2

针对于一个无符号的( 0 - 2^32 ）我们溢出的时候一般都看C位，有符号的就是看O位了，在有无符号的情况下，NOT(a) + b 产生的C/O位与a - b产生的C位总是保持一致的

```
//无符号的状态
NOT(a) = 2^32 - a - 1
2^32 - a - 1 + b > = 2 ^ 32  如果大于等于2^32的时候就溢出了 开始置C位
b >= a + 1
b > a
a - b 置C
```

```
//有符号的状态
NOT(a) = -a - 1
（a>=0    b<0   a-b >= 2^31） /  (a<=0    b>0   a-b < - 2^31)
（-a<=0    b<0   -a-1+b < -2^31） /  (-a-1>=-1    b>0   -a-1+b >= 2^31)
（-a - 1<0    b<0   -a-1+b < -2^31） /  (-a-1>0    b>0   -a-1+b >= 2^31)
等价于NOT(a) + b 置O
```

VMP中 eflag实现的过程：

![image-20210109193115178](VMP/image-20210109193115178.png)

因为上面确定的原则所以如果想要产生真正的eflags我们需要进行两次的pushfd的操作，第一次去取过程化的eflags的操作，第二次去取结果化的eflags的操作，两次的结果进行OR就是我们需要的真正的eflags，所以我们可以对pushfd这个指令进行追踪，这里对eflags的保存还有一个指令就是lahf

![image-20210109194143322](VMP/image-20210109194143322.png)

所以我们把pushfd和lahf都打出来，但是在vmp中一般的lahf都是做混淆的，很少使用大部分的情况就是pushfd

比如说这样的情况我们lahf做完操作之后对我们的eax进行重新的赋值，那么lahf就没有意义了

![image-20210109194917549](VMP/image-20210109194917549.png)

输出的结果：

![image-20210109194059778](VMP/image-20210109194059778.png)

看到这么多的pushfd主要是因为有加减运算的那些，就会产生一次pushfd，但是我们只关心产生z位的那个pushfd

```
00482F8B        pushfd
00436418        pushfd
0046639B        pushfd
0041F1F4        pushfd
00420A7E        pushfd
00429CF9        pushfd
0048A175        pushfd
0046EBD4        pushfd
00421087        pushfd
0042B5BE        pushfd
00464DE9        pushfd
0047D936        pushfd
00421729        pushfd
0048CBC9        pushfd
004335A8        pushfd
0048C880        pushfd
00484710        pushfd
```

测试一下，是对的，vmp主要根据pushfd进行操作

![image-20210109200951383](VMP/image-20210109200951383.png)

通过二分法可以定位一下影响我们zf的那个pushfd在哪里，修改后的结果：所以pushfd可以控制结果的流程

![image-20210109201353794](VMP/image-20210109201353794.png)

因为像pushfd很多，我们如果出现很多的情况的时候，就不能这么二分法慢慢去找，很浪费时间，所以我们可以根据规则，比如我们pushfd之后我们需要 ，与操作，移位的操作，假设这个指令是and eax,0x40 但是我们不知道第二个操作数他是立即数还是寄存器还是内存，或者说他是第一个操作数还是第二个操作数我们是未知的，所以我们要写一个规则通用的：

```c
//判断是什么寄存器
DWORD get_reg(x86_reg reg) {
	switch (reg) {
	case X86_REG_EAX:
		return regs.regs.r_eax;
	case X86_REG_AX:
		return regs.regs.r_eax & 0xffff;
	case X86_REG_AH:
		return (regs.regs.r_eax >> 8) & 0xff;
	case X86_REG_AL:
		return regs.regs.r_eax & 0xff;
	case X86_REG_ECX:
		return regs.regs.r_ecx;
	case X86_REG_CX:
		return regs.regs.r_ecx & 0xffff;
	case X86_REG_CH:
		return (regs.regs.r_ecx >> 8) & 0xff;
	case X86_REG_CL:
		return regs.regs.r_ecx & 0xff;
	case X86_REG_EDX:
		return regs.regs.r_edx;
	case X86_REG_DX:
		return regs.regs.r_edx & 0xffff;
	case X86_REG_DH:
		return (regs.regs.r_edx >> 8) & 0xff;
	case X86_REG_DL:
		return regs.regs.r_edx & 0xff;
	case X86_REG_EBX:
		return regs.regs.r_ebx;
	case X86_REG_BX:
		return regs.regs.r_ebx & 0xffff;
	case X86_REG_BH:
		return (regs.regs.r_ebx >> 8) & 0xff;
	case X86_REG_BL:
		return regs.regs.r_ebx & 0xff;
	case X86_REG_ESP:
		return regs.regs.r_esp;
	case X86_REG_SP:
		return regs.regs.r_esp & 0xffff;
	case X86_REG_EBP:
		return regs.regs.r_ebp;
	case X86_REG_BP:
		return regs.regs.r_ebp & 0xffff;
	case X86_REG_ESI:
		return regs.regs.r_esi;
	case X86_REG_SI:
		return regs.regs.r_esi & 0xffff;
	case X86_REG_EDI:
		return regs.regs.r_edi;
	case X86_REG_DI:
		return regs.regs.r_edi & 0xffff;
	case X86_REG_EIP:
		return regs.regs.r_eip;
	case X86_REG_EFLAGS:
		return regs.regs.r_efl;
	default:
		__asm int 3
	}
}

DWORD read_op(cs_x86_op op)
{
	switch (op.type)
	{
	case X86_OP_IMM: //立即数
		return op.imm;
	case X86_OP_REG: //寄存器
		return get_reg(op.reg);
	case X86_OP_MEM: //内存地址
	{
		DWORD addr = get_mem_addr(op.mem);
		DWORD val = 0;
		uc_mem_read(uc, addr, &val, op.size);
		return val;
	}
	}
}
```

```c
!strcmp(insn->mnemonic,"pushfd")||(!strcmp(insn->mnemonic,"and") && (read_op(insn->detail->x86.operands[0])) == 0x40 || (read_op(insn->detail->x86.operands[1])) == 0x40)||(!strcmp(insn->mnemonic,"shr") && read_op(insn->detail->x86.operands[1]) == 0x6)
```

![image-20210109210532686](VMP/image-20210109210532686.png)

```asm
00429CF9        pushfd
0046EBCF        and ecx, eax
00421082        shr eax, cl
```

可以看到现在的这个位置ecx相当于我们说的一个zf(0x40)的位置的一个判断，eax是我们的eflags

![image-20210109210842653](VMP/image-20210109210842653.png)

shr eax,cl cl是0x6

![image-20210109210949941](VMP/image-20210109210949941.png)



**开启优化的版本：**

![image-20210109211413589](VMP/image-20210109211413589.png)

dump scanf到printf的位置，因为我们的cmp的自定义的函数已经内敛到我们的main中所以定位不到入口和出口的位置

```asm
0046B6F3        pushfd
00464E7F        pushfd
00439BD1        pushfd
0043EB08        pushfd
0041EFCF        pushfd
004818A8        pushfd
0041F922        cmp esp, edx
0041F927        and edx, ecx
00476992        pushfd
00421D22        shr eax, cl
00421D2D        pushfd
0043F7E5        pushfd
00426A2B        mov dl, al
00433BAB        pushfd
0048C281        pushfd
00469322        pushfd
0044F6AF        pushfd
00461644        pushfd
00406D91        pushfd
00432DDE        pushfd
00484E50        pushfd
0043E3D4        pushfd
0047D76A        pushfd
004292FE        pushfd
00424EF5        pushfd
004847BE        pushfd
00489C89        pushfd
00484F54        rol al, cl
004422F1        pushfd
00410E26        pushfd
0046308B        mov dx, ax
0041E2FA        pushfd
0041C103        pushfd
00451729        and edx, ecx
004725D1        mov dword ptr [esi + 4], edx
004725D4        pushfd
00483E8C        mov eax, dword ptr [esi]
00483E8E        add cl, al
00483E9B        shr eax, cl
00483EA9        pushfd
0046D448        pushfd
0041074E        pushfd
00470720        pushfd
0043A841        pushfd
0043A284        pushfd
004313A6        rcr cx, cl
00451B58        pushfd
0041D33F        pushfd
0040AFE2        pushfd
0040DC15        pushfd
004448CD        pushfd
0042FC1E        pushfd
0041DB6F        pushfd
004730D6        pushfd
00448E3F        pushfd
0048A934        pushfd
0044F92B        and eax, ecx
0044F939        pushfd
00454033        shr edx, cl
004548C6        pushfd
0047305F        pushfd
00476B19        pushfd
0044F99F        pushfd
004473A7        pushfd
00483B58        pushfd
004092C0        pushfd
00458B9F        pushfd
00428600        pushfd
004811BF        movzx ecx, byte ptr [ebp]
```

```asm
0041F927        and edx, ecx
00421D22        shr eax, cl

00451729        and edx, ecx
00483E9B        shr eax, cl

0044F92B        and eax, ecx
00454033        shr edx, cl
```

下断跑一下，看一下加了优化后的流程，总结一下流程图第一次，但是我们不知道0x421d22的分支上是否还有别的jcc

![image-20210109221654120](VMP/image-20210109221654120.png)

```asm
004695F5        and ecx, eax
0043CB41        shr eax, cl
```

![image-20210109222524671](VMP/image-20210109222524671.png)



```asm
00473A47        and ecx, edx
0046B07D        shr eax, cl
```

![image-20210110143926167](VMP/image-20210110143926167.png)

```asm
0040A211        and ecx, eax
00431FB2        shr eax, cl
```



![image-20210110144739132](VMP/image-20210110144739132.png)



### 侧信道攻击

我们可以把我们的程序的流程指令的个数都打印出来，我们如果遇见像分支一类的情况下，我们就可以发现指令的个数会有明显的一个改变的形式，其实这个东西也不是很重要，因为如果我们作为正向的开发肯定原则不会这么去写代码，一定会加上一些加密或者是其他的原则让这种方式不会成功

但是还是要看一下测信道攻击是什么情况：

![image-20210110152512270](VMP/image-20210110152512270.png)

如果第一个不是正确的数值路径就会变少：

![image-20210110152551746](VMP/image-20210110152551746.png)

所以可以通过这种方式来侧信道攻击从而达到一个确定最后的结果是什么的情况，但是这种问题很容易被加密的算法等等所干掉，让这种办法不可以进行，因为计算机的运算速度不满足（只是说一下可以有这个方式）



### 污点分析

被污染的数据，在代码的执行指定的时间点上面，因为输入的数据的不同而可能产生不同取值的数据

简要的来说，因为我们的输入去导致了一个数值的修改，我们的这个数值就叫做被污染，那么这个污染的数值如果对另一个数值进行写的操作，那么另一个数值也就被污染了，如果这个数值被其他数值写操作了，那么这个数值就解除污染，这个有点像游戏分析中的数据追踪，去获取谁访问了该数值，然后追踪找到最后的基址的方式是差不多的

污染传播的一般原则：

- 在一个指令中，如果有至少一个读位置是污染的，就将所有的写位置污染
- 在一条指令中，如果所有的读位置都是非污染的，那么就将所有的写位置去污染

但是比如这些原则来说针对于汇编的一些指令，有些是不满足这个规则的，比如说xchg eax,ecx 我们对两个寄存器都进行了一个读的操作，并且两个寄存器都有写操作，按照我们的原则来说我们的两个寄存器就都会进行污染，但实际上其实就是一个寄存器被污染了，所以像这些指令来说，我们需要拿出来单独的做规则



假设一个例子代码

我们的push指令，push eax为例子，我们对eax进行读的操作，对堆栈的内存进行写的操作，要写不同的指令的污点的代码

```C

if (!strcmp(insn->mnemonic, "push"))
{//push
	cs_x86_op op = x86.operands[0];
	do_taint_sp_push(op);
	return g_taint_handled;
}

/* ------------------------------------------------------------------- */
inline static void do_taint_sp_push(cs_x86_op& op)
{
	DWORD esp_after = regs.u[reg_transfer_table[X86_REG_ESP]] - 4;

	switch (op.type)
	{
	case X86_OP_MEM:
	{
		DWORD addr = get_mem_addr(op.mem);
		if (is_addr_tainted(addr) || is_addr_tainted(addr + 1) || is_addr_tainted(addr + 2) || is_addr_tainted(addr + 3)) {
			for (int i = 0; i < 4; i++)
				taint_addr(esp_after + i);
		}
		else {
			for (int i = 0; i < 4; i++)
				untaint_addr(esp_after + i);
		}
	}
	break;

	case X86_OP_REG:
	{
		x86_reg reg = op.reg;
		if (is_reg_tainted(reg)) {
			for (int i = 0; i < 4; i++)
				taint_addr(esp_after + i);
		}
		else {
			for (int i = 0; i < 4; i++)
				untaint_addr(esp_after + i);
		}
	}
	break;

	case X86_OP_IMM:
		for (int i = 0; i < 4; i++)
			untaint_addr(esp_after + i);
		break;

	default:
		__asm int 3
	}
}
```

污点分析最好的一个好处就是因为我们对一个污染源做完操作之后他一定会有一个终止的位置，使用的最佳的位置，比如说我要比较cmp举例子到最后的测试的时候看到大部分都是进行要给 movzx  xxx,xxx 这样的指令去写入寄存器 (最后一次的时候) 进行比较

















