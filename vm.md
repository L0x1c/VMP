## VM 
主要是自己看加密与解密4的时候vm部分的笔记

VM的存在，主要是将可执行代码转换为字节码指令系统的代码，达到原有的指令不会被容易的逆向以及篡改，流程图：

![1.png](https://i.loli.net/2020/12/29/SK2MgXf7u9pi8Fl.png)

VStartVM部分初始化虚拟机，VMDispatcher调度Handler，每个Handler就是CPU支持的一条指令，ByteCode是CPU执行的二进制代码

### 启动框架

VM的启动框架和调用约定，每种代码执行模式都需要有启动框架和调用约定，在C语言中，在进入main函数之前，会有一些C库添加的启动函数，它们负责对栈区，变量进行初始化，在main函数执行完毕之后再收回控制权，这就叫做启动框架，而C语言 CALL，StdCall等约定就是调用约定，它们规定了传参方式和堆栈平衡方式，同样，对与VM虚拟机，我们也需要有一种启动框架和调用约定，来保证虚拟指令的正确执行以及虚拟和现实代码之间的切换

我们的VStartVM将环境压入栈后生成一个VMDispatcher标签，当我们的Handler执行完在跳回这里，形成循环

VStartVM将所有的寄存器压入堆栈，esi指向字节码的起始地址，ebp指向真实的栈，edi指向VMContext，esp的值进行了运算就是我们所说的VM使用的栈地址，VM的环境结构和栈都放在了当前栈的200h处，如果接近自己存放的数据，我们就直接读取字节码，读取一个字节，在JUMP表中寻找对应的Handler跳转过去执行

```asm
push opcode地址
jmp VStartVM
```

```asm
VStartVM:
	push eax
	push ebx
	push ecx
	push edx
	push esi
	push edi
	push ebp
	pushfd 
	mov esi,[esp+0x20] ;字节码开始位置
	mov ebp,esp ;真实栈
	sub esp,0x200
	mov edi,esp ;VMContext
	sub esp,0x40 ;这时的esp就是VM用的堆栈了
vBegin:
...
	jmp VMDispatcher
VMDispatcher:
	movzx	eax,byte ptr [esi] ;获得bytecode
	lea esi,[esi+1] 
	jmp dword ptr [eax*4+JUMPADDR] ;跳到handler执行处
```

```
  堆栈低地址...
  ... --------
  ...    |
  ... 0x40字节的空间，VM使用的堆栈空间（不一定非要0x40大小）
  ...    |
  ... --------                               edi指向这里(VMcontext)
  ...    |
  ...  0x200字节的空间，用于存放VMcontext
  ...    |
  ... --------                              
  flags  保存的标志寄存器                     ebp指向这里
  ebp --------
  edi     |
  esi     |
  edx    保存的寄存器
  ecx     |
  ebx     |
  eax --------
  伪代码开始地址(虚拟机参数)                  esi指向这里(伪代码地址)
  ...
  堆栈高地址...
```

VMcontext：虚拟环境结构，存放了一些需要用的值

```c
struct  VMContext{
  DWORD	v_eax;
  DWORD v_ebx;
  DWORD v_ecx;
  DWORD v_edx;
  DWORD v_esi;
  DWORD v_edi;
  DWORD v_ebp;
  DWORD v_efl;
}
```

VStartVM将所有的寄存器都压入了栈，栈达到平衡后，才开始执行真正的代码

#### 平衡栈vBegin

Handle vBegin代码：

```asm
vBegin:
	mov eax,dword ptr [ebp]
	mov [edi+0x1c],eax						;v_efl
	add ebp,4											
	mov eax,dword ptr [ebp]
	mov [edi+0x18],eax						;v_ebp
	add ebp,4											
	mov eax,dword ptr [ebp]
	mov [edi+0x14],eax						;v_edi
	add ebp,4											
	mov eax,dword ptr [ebp]
	mov [edi+0x10],eax						;v_esi
	add ebp,4												
	mov eax,dword ptr [ebp]
	mov [edi+0xC],eax							;v_edx
	add ebp,4											
	mov eax,dword ptr [ebp]
	mov [edi+0x8],eax							;v_ecx
	add ebp,4											
	mov eax,dword ptr [ebp]
	mov [edi+0x4],eax							;v_ebx
	add ebp,4											
	mov eax,dword ptr [ebp]
	mov [edi],eax									;v_eax
	add ebp,4											
	add ebp,4											;push字节码地址
	jmp	VMDispatcher
```

为了避免改写VMcontext的内容，结构覆盖，设计了Handler vCheckESP：

#### 检查vCheckESP

```asm
vCheckESP:
	lea	eax,dword ptr [edi+0x100]
	cmp eax,ebp	;比较了ebp和中间位置的地址
	jl	VMDispatcher	;小于就跳转
	mov edx,edi
	mov ecx,esp
	sub ecx,edx
	push esi
	mov esi,esp
	sub esp,0x60
	mov edi,esp
	push edi
	sub esp,0x40
	cld	 ;递增的方式
	rep movsb ;复制
	pop edi
	pop esi
	jmp VMDispatcher
```

涉及到栈的Handler执行后跳转到vCheckESP，判断esp是否接近VMContext，接近就提升

### Handler设计

handler分两大类：

1. 辅助handler，指一些更重要的、更基本的指令，如堆栈指令
2. 普通handler，指用来执行普通的x86指令的指令，如运算指令

辅助handler除了VBegin这些维护虚拟机不会导致崩溃的handler之外，就是专门用来处理堆栈的handler了

#### 辅助Handler

```asm
vPushReg32:
    mov eax, dword ptr [esi] ;从字节码中得到VMContext中的寄存器偏移
    add esi, 4
    mov eax, dword ptr [edi+eax] ;得到寄存器的值
    push eax ;压入寄存器
    jmp VMDispatcher

vPushImm32:
    mov eax, dword ptr [esi]
    add esi, 4
    push eax
    jmp VMDispatcher

vPushMem32:
    mov eax, 0
    mov ecx, 0
    mov eax. dword ptr [esp] ;第一个寄存器偏移
    test eax, eax
    cmovge edx, dword ptr [edi+eax] ;如果不是负数则赋值
    mov eax, dword ptr [esp+4] ;第二个寄存器偏移
    test eax, eax
    cmovge ecx, dword ptr [edi+eax] ;如果不是负数则赋值
    imul ecx, dword ptr [esp+8] ;第二个寄存器的乘积
    add ecx, dword ptr [esp+0x0C] ;第三个为内存地址常量
    add edx, ecx
    add esp, 0x10 ;释放参数
    push edx ;插入参数
    jmp VMDispatcher

vPopReg32:
    mov eax, dword ptr [esi] ;得到reg偏移
    add esi, 4
    pop dword ptr [edi+eax] ;弹回寄存器
    jmp VMDispatcher

vFree:
    add esp, 4
    jmp VMDispatcher
```

用个图来举个例子：正常的x86指令的Handler

![2.png](https://i.loli.net/2020/12/29/vbeDYUOtKAud2ap.png)

指令是由我们的普通Handler处理，源操作数，目的操作数，寄存器都是由辅助Handler处理，这么设计的好处就是，假设我们的add指令有add reg,imm add reg,reg add reg,mem ... 那么我们如果有辅助handler和普通handler就不需要每一个模式都写一个handler，我们先把操作数都交给辅助handler处理，那么执行普通handler的时候，操作数已经变成了立即数存放了，直接用立即数就可以了，将辅助指令和普通指令配合起来来一起完成x86指令到伪指令的转换

#### 普通Handler

实现一个vadd：

```asm
vadd:
    mov eax, [esp+4] ;取源操作数
    mov ebx, [esp] ;取目的操作数
    add ebx, eax
    add esp, 8 ;平衡堆栈
    push ebx ;压入堆栈
```

指令：

```asm
add esi, eax
```

转换代码：

```asm
vPushReg32 eax_index    ;eax在VMContext下的偏移
vPushReg32 esi_index
vadd
vPopReg32 esi_index
--------------------------------------------------------------------
vPushReg32:
    mov eax, dword ptr [esi] ;从字节码中得到VMContext中的寄存器偏移
    add esi, 4
    mov eax, dword ptr [edi+eax] ;得到寄存器的值
    push eax ;压入寄存器
    jmp VMDispatcher
---------------------------------------------------------------------
vPopReg32:
    mov eax, dword ptr [esi] ;得到reg偏移
    add esi, 4
    pop dword ptr [edi+eax] ;弹回寄存器
    jmp VMDispatcher
```

指令：

```asm
add	esi,1234
```

转换代码：

```asm
vPushImm32 1234
vPushReg32 esi_index
vadd
vPopReg32	esi_index
---------------------------------------------------------------------
vPushImm32:
    mov eax, dword ptr [esi]
    add esi, 4
    push eax
    jmp VMDispatcher
---------------------------------------------------------------------
vPushReg32:
    mov eax, dword ptr [esi] ;从字节码中得到VMContext中的寄存器偏移
    add esi, 4
    mov eax, dword ptr [edi+eax] ;得到寄存器的值
    push eax ;压入寄存器
    jmp VMDispatcher
---------------------------------------------------------------------
vPopReg32:
    mov eax, dword ptr [esi] ;得到reg偏移
    add esi, 4
    pop dword ptr [edi+eax] ;弹回寄存器
    jmp VMDispatcher
---------------------------------------------------------------------    
vadd:
    mov eax, [esp+4] ;取源操作数
    mov ebx, [esp] ;取目的操作数
    add ebx, eax
    add esp, 8 ;平衡堆栈
    push ebx ;压入堆栈    
```

指令：  原操作数是一个内存数，内存数真实结构[ imm + reg * scale + reg2 ]

```asm
add	esi,dword	ptr	[401000]
```

转换代码 :

```asm
vPushImm32 401000
vPushImm32 1	;scale
vPushImm32 -1	;reg2_index
vPushImm32 -1	;reg_index
vPushMem32	;压入内存地址
vPushReg32 esi_index
vadd
vPopReg32	esi_index
---------------------------------------------------------------------  
vPushImm32:
    mov eax, dword ptr [esi]
    add esi, 4
    push eax
    jmp VMDispatcher
---------------------------------------------------------------------  
vPushMem32:
    mov eax, 0
    mov ecx, 0
    mov eax, dword ptr [esp] ;第一个寄存器偏移
    test eax, eax
    cmovge edx, dword ptr [edi+eax] ;如果不是负数则赋值
    mov eax, dword ptr [esp+4] ;第二个寄存器偏移
    test eax, eax
    cmovge ecx, dword ptr [edi+eax] ;如果不是负数则赋值
    imul ecx, dword ptr [esp+8] ;第二个寄存器的乘积
    add ecx, dword ptr [esp+0x0C] ;第三个为内存地址常量
    add edx, ecx
    add esp, 0x10 ;释放参数
    push edx ;插入参数
    jmp VMDispatcher
---------------------------------------------------------------------  
vadd:
    mov eax, [esp+4] ;取源操作数
    mov ebx, [esp] ;取目的操作数
    add ebx, eax
    add esp, 8 ;平衡堆栈
    push ebx ;压入堆栈 
---------------------------------------------------------------------
vPopReg32:
    mov eax, dword ptr [esi] ;得到reg偏移
    add esi, 4
    pop dword ptr [edi+eax] ;弹回寄存器
    jmp VMDispatcher
```

无论什么形式，都可以使用vadd来执行，只是使用了不用的栈Handler

#### 标志位的问题

标志的问题一般来说，有的指令是设置标志位，有的指令是判断标志位，所以，应该在相关Handler执行之前保存标志位，在相关Handler执行后恢复标志位，比如stc命令是让标志位置1

```asm
VStc:
    push [edi+0x1C]  ;EFL
    popfd	;返回到EFL
    stc		;改变
    pushfd	;EFL到堆栈
    pop [edi+0x1C]	;返回给EFL
    jmp VMDispatcher
```

#### 转移指令

esi指针相当于我们真实CPU中的eip寄存器，可以通过改写esi寄存器的值来改变流程

jmp：

``` asm
vJmp:
    mov esi, dword ptr [esp] ;[esp]指向要跳转到的地址
    add esp, 4
    jmp VMDispatcher
```

jcc条件转移指令和comvcc条件传输指令高度匹配

| 条件跳转指令               | 条件传输指令 |
| -------------------------- | ------------ |
| jne //不等于则跳转         | cmovne       |
| ja //无符号大于则跳转      | comva        |
| jae //无符号大于等于则跳转 | cmovae       |
| jb //无符号小于则跳转      | cmovb        |
| jbe //无符号小于等于则跳转 | cmovbe       |
| je //等于则跳转            | cmove        |
| jg //有符号大于则跳转      | cmovg        |

所有的跳转都有条件传输指令

```asm
vJne:
    cmovne esi, [esp]
    add esp, 4
    jmp VMDispatcher

vJa:
    cmova esi, [esp]
    add esp, 4
    jmp VMDispatcher

vJae:
    cmovae esi, [esp]
    add esp, 4
    jmp VMDispatcher

vJb:
    cmovb esi, [esp]
    add esp, 4
    jmp VMDispatcher

vJbe:
    cmovbe esi, [esp]
    add esp, 4
    jmp VMDispatcher

je:
    cmove esi, [esp]
    add esp, 4
    jmp VMDispatcher

jg:
    cmovg esi, [esp]
    add esp, 4
    jmp VMDispatcher
```

```asm
jecxz: //JECXZ(ECX 为 0 则跳转)
		mov ecx,[edi+8]
		test ecx,ecx
		cmovz esi,eax   //JZ   ;为 0 则跳转
		add esp,4
		jmp VMDispatcher
```

#### 转移指令另一种实现

如果按照上面的做法来进行模拟跳转比较简单，所以可以在模拟转移指令时候判断标示位

![3.png](https://i.loli.net/2020/12/29/SzZYTV6CFjL9vqt.png)

如果知道了标志位所在的位置，就可以模拟条件跳转

```asm
vJAE:
		push [edi+0x1C]
		pop	eax
		and eax,1
		cmove	esi,[esp]
		add	esp,4
		jmp	VMDispatcher
```

也可以调用指令：

```asm
vPush jumptoaddr ;跳转的地址
vJae
```

这个指令首先得到标识位，然后1和eax = EFL 进行and，comve用于判断ZF标志是否为0，如果是0就改变esi指向，JAE指令只判断我们的CF位

```asm
vJBE:
		push [edi+0x1C]
		pop	eax
		and eax,0x41	;1001Bh
		cmp eax,0x41	;如果小于等于则转移(CF=1 || ZF=1)
		cmove esi,[esp]	
		add	esp,4
		jmp VMDispatcher
```

其他同理

#### call指令

首先，虚拟机设计为只在一个堆栈层次上运行

```asm
    mov eax, 1234
    push 1234
    call anotherfunc
theNext:
    add esp, 4
```

其中第1、2、4条指令都是在当前堆栈层次上执行的，而call anotherfunc是调用子函数，会将控制权移交给另外的代码，这些代码是不受虚拟机控制的，所以碰到call指令，必须退出虚拟机，让子函数在真实CPU中执行完毕后再交回给虚拟机执行下一条指令

```asm
    push theNext
    jmp anotherfunc
```

如果想在推出虚拟机后让anotherfunc这个函数返回后再次拿回控制权，可以更改返回地址，来达到继续接管代码的操作，在一个地址上写上这样的代码:

```asm
theNextVM:
    push theNextByteCode
    jmp VStartVM
```

这是一个重新进入虚拟机的代码，theNextByteCode代表了theNext之后的代码字节码。只需将theNext的地址改为theNextVM的地址，即可完美地模拟call指令了。当虚拟机外部的代码执行完毕后ret回来的时候就会执行theNextVM的代码，从而使虚拟机继续接管控制权

```asm
vcall:
    push all vreg ;所有虚拟寄存器
    pop all reg ;弹出到真实寄存器中
    push 返回地址 
    push 要调用的函数的地址
    retn
```

#### retn指令

retn在虚拟机里当作一个退出函数，retn一种是不带参数的，一种带参数

```asm
retn
retn	4
```

第一种得到esp存放的返回地址，释放返回地址的栈并跳转到返回地址，第二种释放返回地址的栈时再释放操作数空间

```asm
vRetn:
		xor eax,eax	
		mov ax,word ptr [esi]	;retn的操作数是WORD的，所以值为0xFFFFh
		add	esi,2
		mov ebx,dword ptr [ebp] ;得到要返回的地址
		add	ebp,4	;释放空间
		add ebp,eax	;如果有操作数，释放
		push ebx	;压入返回地址
		push ebp	;压入栈指针
    push [edi+0x1c]
    push [edi+0x18]
    push [edi+0x14]
    push [edi+0x10]
    push [edi+0x0c]
    push [edi+0x08]
    push [edi+0x04]
    push [edi]
    pop	eax
    pop	ebx
    pop ecx
    pop edx
    pop esi
    pop edi
    pop ebp
    popfd
    pop esp	;将栈指针还原到esp中，VM_Context自动销毁
    retn
```

#### 不可模拟指令

不能识别的指令为不可模拟指令，我们只能vcall退出虚拟机，执行这个指令

### VC++异常处理

VC 7编译器生成的栈帧布局，Scopetable是一个记录（record）的数组，每个record描述了一个＿try块，以及块之间的关系

![4.png](https://i.loli.net/2020/12/29/QJCVMkix7K4qYac.png)

```c
sturt _SCOPETABLE_ENTRY
{
    DWORD EnclosingLevel;
    void* FilterFunc;
    void* HandlerFunc;
}
```

Stack frame: 栈帧, 被某个函数使用的一段堆栈段. 通常包含函数参数, 返回地址, 保存的寄存器现场, 局部变量和其他一些特定于这个函数的数据. 在x86(和大多数的其他架构)上, 调用者与被调用者的栈帧是相邻的.

Frame pointer: 栈帧指针, 一个指向栈帧里固定地址的寄存器或其他变量. 通常栈帧里的所有数据都是用它来作相对寻址. 在x86上它通常是ebp而且通常是指向返回地址的下面.

Object: 对象, 一个(C++)类的实例.

Unwindable Object: 可展开对象, 一个被指定为auto存储级别(auto storage-class)的局部对象, 它被分配在栈里, 而且在离开作用范围之后被销毁.

Stack Unwinding: 栈展开, 自动销毁上面那些对象, 在因异常使程序流离开它们的作用范围时发生

在C或C++程序里可以使用两种异常:

SEH(Structured Exception Handling) 异常, 结构化异常处理异常. 就是平常说的Win32或系统异常

C++异常(有时称为"EH"). 在SEH之上实现. C++异常允许抛出和捕获任意类型



Whidbey(MSVC 2005)编译器在SEH帧里添加了一些缓冲区溢出的保护措施：

![5.png](https://i.loli.net/2020/12/29/HOFkBQe2WJvCN4A.png)

```c
struct _EH4_SCOPETABLE 
{
	DWORD GSCookieOffset;
	DWORD GSCookieXOROffset;
	DWORD EHCookieOffset;
	DWORD EHCookieXOROffset;
	_EH4_SCOPETABLE_RECORD ScopeRecord[1];
};

struct _EH4_SCOPETABLE_RECORD 
{
	DWORD EnclosingLevel;
	long (*FilterFunc)();
	union 
  {
		void (*HandlerAddress)();
		void (*FinallyFunc)();
	};
};
```
