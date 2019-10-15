原文为360的安全专家很久之前的一篇文章，偶然发现后收获不少。特以笔记的形式记录下来, 便于加深理解。

<font color="red">
Intel有公开的[指令集格式文档](http://www.intel.com/products/processor/manuals/)，你需要的是[第二卷的上半部分](http://www.intel.com/Assets/PDF/manual/253666.pdf)，指令集从A到M。这篇文档的难度超出一般人想象，里面有众多晦涩的标识、与硬件紧密相关的介绍，拿到这后，即使直接翻到目录的CALL 指令一节，也不见得能够弄清楚。不相信？我们就翻到那里看看：	

![11](http://mailshark.nos-jd.163yun.com/document/static/AC910F64498C34D0903C68D516C5969E.png)
</font></p>

这里注意一个细节, 即第四列64-bit Mode。能看到最后一行是Invalid, 这意味着在64位环境下该指令无效, 由于段寄存器这个概念在64位中已经不再使用, 猜测是这个缘由。

<p><font color="red">
虽然很明确的列出，第一列是指令的二进制形式，第二列是指令的汇编形式，但是面对着 E8 cw, FF/2这样的标识，一样不知道究竟对应的二进制格式是什么样的。
</font></p>

<p><font color="red">
那好，我们就从理解这些标识开始。文档向前翻，有一个专门的节(3.1.1 Instruction Format）讲述这些标识的含义。这里抽出其中两个用得着的翻译一下：
</font></p>

<p><font color="red">
表格中的“Opcode”列列出了所有的所有可能的指令对应的二进制格式。有可能的话，指令代码使用十六进制显示它们在内存当中的字节。除了这些16进制代码之外的部分使用下面的标记：
</font></p>

注意这里用了”所有“这个字眼。以当前为例, 意味着所有的call指令都能用上述规则来表达。

<p><font color="red">
**cb, cw, cd, cp, co, ct**  — opcode后面跟着的一个1字节(cb)，2字节(cw)，4字节(cd)，6字节 (cp)，8字节(co) 或者 10字节(ct) 的值。这个值用来表示代码偏移地址，有可能的话还包括代码段寄存器的值。
</font></p>

这里只是举例。call指令后只有cw和cd, 和其cb,cp,co,ct无关。

<p><font color="red">
**/digit**  — digit为0到7之间的数字，表示指令的  **ModR/M byte** 只使用  **r/m字段**作为操作数，而其**reg字段**作为opcode的一部分，使用digit指定的数字。
</font></p>

MODR/M规则要牢牢记住。 
7 6   Mod
5 4 3 reg/opcode (此时digit被认为是opcode的一部分)
2 1 0 R/M

<p><font color="red">
红字部分不知道什么含义？没关系，我们先不看它。对于cb/cw之类的，基本上能够简单看明白其中的一些指令含义了：
</font></p>

<p><font color="red">
E8 cw 的含义是：字节 0xE8 后面跟着一个2字节操作数表示要跳转到的地址与当前地址的偏移量。  
E8 cd 的含义是：字节 0xE8 后面跟着一个4字节的操作数表示要跳转的地址与当前地址的偏移量。  
9A cd 的含义是：字节 0x9A 后面跟着一个6字节的操作数表示要跳转的地址和代码段寄存器的值。
<p><font color="red">

关于9A后跟6字节是有点疑惑的。但是由于64位不考虑9A, 所以先忽略作罢。
E8 cw也是忽略的。因为正常工作情况下cpu都是保护模式使用4字节。
所以这里只有E8 cd, 即跟随了4字节的偏移量。

<p><font color="red">
那么，同样的0xE8开头的指令，CPU如何区分后面的操作数是2字节还是4字节？答案是和CPU的模式有关，在实模式下，0xE8接受2字节操作数，而32位保护模式下接受4个字节，64位保护模式下同样接受4字节，同时需要对该操作数进行带符号扩展。
</font></p>

[https://zh.wikipedia.org/wiki/X86-64](https://zh.wikipedia.org/wiki/X86-64)。
按照维基百科上的理解，现有的X86_64是AMD基于IA_32架构上做出的扩展, 现有的X86_64并不是基于IA64的, IA64做出来之后貌似无人问津。
而实模式和保护模式google到的结过是这样的, CPU开始加电是实模式, 正式开始工作就是保护模式了。所以我们这里统一认为E8在64位下后跟4字节。

<p><font color="red">
因此，CALL指令的前两种格式是：E8 xx xx xx xx，和 9A xx xx xx xx xx xx。一个是5字节长，一个是7字节长。其实E8 那种，就是我们在汇编指令里面写 CALL lable之后产生的，最常见的CALL指令。
</font></p>

9A指令之前已经描述过在64位不可用。所以这里我们就忽略了。这里只认E8 xx xx xx xx。

<p><font color="red">
然后是下面的FF /2。这个是0xFF字节后面跟上一个blablabla的东西。这个blablabla的东西是什么呢？要解释这个，首先需要知道红字标出来的部分，即ModR/M是什么东西。
</font></p>

<p><font color="red">
这个要先回到最基本的一个问题：IA32的指令格式。

![IA32](http://mailshark.nos-jd.163yun.com/document/static/4AD61D6E6E559BC37A109664658B70B8.png)
</font></p>
<p><font color="red">
其中每个部分是什么含义呢？
</font></p>
<p><font color="red">
首先是指令前缀。有印象的应该记得当年学习微机原理的时候提到过得循环前缀 repnz/repne，这个前缀就是被编码在指令的前面部分的。每个前缀最多一个字节，一条指令最多4个前缀。
</font></p>

关于指令前缀摘抄自X86的维基百科。
[https://zh.wikipedia.org/wiki/X86](https://zh.wikipedia.org/wiki/X86)
分为4组，每组用1个字节编码。每组在指令中至多指定1个前缀值。4组的顺序可以任意。

-   第1组锁与重复（Lock and repeat）
    -   锁（LOCK）编码为：F0H。用于互斥访问共享内存的操作。
    -   非零时重复（REPNE/REPNZ）编码为：F2H。用于字符串操作指令。
    -   为零时重复（REP/REPE/REPZ）编码为：F3H。用于字符串操作指令。
-   第2组
    -   段覆盖（Segment override）：CS、SS、DS、ES、FS、GS的段覆盖前缀的编码分别是2EH、36H、3EH、26H、64H、65H.
    -   分支提示（Branch hints），用于条件分支指令J_cc_。提示分支不发生编码为2EH；提示分支发生编码为3EH。
-   第3组操作数长度覆盖（Operand-size override）编码为66H。用于在16位与32位操作数切换。
-   第4组地址长度覆盖（Address-size override）编码为67H.用于在16位与32位地址切换。


<p><font color="red">
然后是指令代码（opcode），这部分标识了指令是什么。这个是指令当中唯一必需的部分。前面例子当中的 0xE8，0xFF都是opcode。
</font></p>
<p><font color="red">
再后面就是我们要重点关心的 ModR/M字段了，还有和它密切相关的SIB字节。手册2.1.3当中有对于它们的详细描述。
</font></p>

<p><font color="red">
许多指令需要引用到一个在内存当中的值作为操作数，这种指令需要一个称为**寻址模式标识字节**（addressing-form specifier byte），或者叫做ModR/M字节紧跟在主opcode后面。ModR/M字节包含下面三个部分的信息：
</font></p>

-   <font color="red">mod（模式）域，连同r/m（寄存器/内存）域共同构成了32个可能的值：8个寄存器和24个寻址模式。</font>
-   <font color="red">reg/opcode（寄存器/操作数）域指定了8个寄存器或者额外的3个字节的opcode。究竟这三个字节用来做什么由主opcode指定。</font>
-   <font color="red">r/m（寄存器/内存）域可以指定一个寄存器作为操作数，或者可以和mod域联合用来指定寻址模式。有时候，它和mod域一起用来为某些指令指定额外的信息。 </font>

个人的理解是这样的:
MOD有四种取值类型
00: 寄存器间接寻址(register indirect addressing) 
01: 寄存器相对寻址或基址相对寻址(register relative addressing)  disp8
10: 寄存器相对寻址或基址相对寻址(register relative addressing)  disp32
11: 寄存器寻址(register addressing)

<p><font color="red">
这一段有些晦涩。其意思解释一下是这样的：一个指令往往需要引用一个在内存当中的值，典型的就是如mov：
</font></p>

<p><font color="red">
MOV eax, dword ptr [123456]
MOV eax, dword ptr  [esi]
</font></p>

将0x123456地址处的内容送入eax(长度为双字)。
将esi寄存器中保存的地址处的内容送入eax(长度为双字)。

<p><font color="red">
这其中的 123456 或者 esi 就是 MOV 指令引用的内存地址，而MOV关心的是这个地址当中的内容。这个时候，需要某种方式来为指令指定这个操作数的类型：是一个立即数表示的地址，还是一个存放在寄存器当中的地址，或者，就是寄存器本身。
</font></p>
 
这一句话点醒了我。它说明了汇编为什么要这么写， 以及为什么有SIB和MODR/M.

<p><font color="red">
这个用来区分操作数类型的指令字节就是 ModR/M，确切的说是其中的5个位，即mod和r/m域。剩下的三个位，可能用来做额外的指令字节。因为，IA32的指令个数已经远超过一个字节所能表示的256个了。因此，有的指令就要复用第一个字节，然后依据ModR/M当中的reg/opcode域进行区分。
</font></p>

重申, MODR/M规则要牢牢记住。 
7 6   Mod
5 4 3 reg/opcode  
2 1 0 R/M

这里的”复用第一个字节“, 应该说的是指令前缀。

<p><font color="red">
现在回头看前面的红字标识的部分，能不能理解 /digit 这种表示法了？
</font></p>

<p><font color="red">
对于SIB的介绍，我们先忽略，看看对于CALL指令的枚举我们已经能做什么了。
</font></p>

<p><font color="red">
CALL指令的表示法：FF /2，是 0xFF 后面跟着一个 /digit 表示的东西。就是说，0xFF后面需要跟一个 ModR/M 字节，ModR/M字节使用 reg/opcode 域 = 2 。那么，reg/opcode = 2 的字节有32个，正如ModR/M的解释，这32个值代表了32种不同的寻址方式。是哪32种呢？手册上面有张表：

![222](http://mailshark.nos-jd.163yun.com/document/static/B2C2154815A7743D501A7F225E9A8B7C.png)
</font></p>

原文说"reg/opcode = 2 的字节有32个", 其实从表里能看到, opcode为任何数(0-7), 对应的可能性都是32种(4*8)。

<p><font color="red">
非常复杂的一张表。现在就看看这张表怎么读。
</font></p>

<p><font color="red">
首先是列的定义。由于 reg/opcode 域可以用来表示opcode，也可以用来表示reg，因此同一个值在不同的指令当中可能代表不同的含义。在表当中，就表现为每一列的表头都有很多个不同的表示。我们需要关心的就是 opcode 这一个。注意看我用红圈圈出来的部分，这一列就是 opcode=2 的一列。而我们需要的 CALL 指令，也就是在这一列当中，0xFF后面需要跟着的内容。
</font></p>

大部分情况下， 我们只关注reg/opcode对应的是opcode的情况。

<p><font color="red">
行的定义就是不同的寻址模式。正如手册所说，mod + R/M域，共5个字节，定义了32种寻址模式。0x10 – 0x17 对应于寄存器寻址。例如指令 CALL dword ptr [eax] ：[eax]寻址对应的是 0x10，因此，该指令对应的二进制就是 FF 10。同理， CALL dword ptr [ebx] 是 FF 13，CALL dword ptr [esi] 是 FF 16，这些指令都是2个字节。有人也许问 CALL word ptr [eax] 是什么？抱歉，这不是一个合法的32位指令。
</font></p>

思考1: 
大学里学过的一点点汇编早特么忘了。[https://blog.csdn.net/zzzyyyyyy66/article/details/80329591](https://blog.csdn.net/zzzyyyyyy66/article/details/80329591)
这里的 call dword ptr [eax] 意思为: 将eax对应的值作为地址，并返回以此开始的4个字节。 然后以这4个字节对应的【内容】为地址，发起调用。
而call eax意思为: 认为当前eax的内容就是地址，直接发起调用。
这里还要纠正另外一个问题, 根据维基百科对于X86的解释, 这里应该是【寄存器间接寻址】， 而不是【寄存器寻址】。

思考2:
行是寻址模式。"Value of ModR/M Byte"实际上描述了ModR/M的8位作为一个整数计算出来的16进制的值。
64位下, 这里实际操作的是rax. 比如FF 10:
32位下: call dword ptr [eax] 
64位下: call qword ptr [rax] 
这里有个例外是0x14使用了SIB, 0x15使用了立即数。这2个并不是寄存器。这个回头会说。
 
<p><font color="red">
0x50-0x57部分需要带一个 disp8，即 8bit 立即数，也就是一个字节。这个是基地址+8位偏移量的寻址模式。例如 CALL dword ptr [eax+10] 就是 FF 50 10 。注意虽然表当中写的是 [eax] + disp8 这种形式，但是并不表示是取得 eax 指向的地址当中的值再加上 disp8，而是在eax上加上disp8再进行寻址。因此写成 [eax+disp8] 更不容易引起误解。后面的disp32也是一样的。这个类型指令是3个字节。
</font></p>

这里0x54是个例外,使用了SIB. 0x50-0x57的范围内限制了理解数是disp8, 那么比如FF 50 10 20 33就会被识别为:

    call qword ptr[rbp+0x10]
    and byte ptr[rbx], dh

很明显cpu识别到10就知道FF这个call指令到此为止了， 从20开始又是一条新的指令。
需要再一次注意这种写法是在进行地址相加而不是值。

<p><font color="red">
0x90 – 0x97部分需要带 disp32，即4字节立即数。这个是基地址+32位偏移量。例如 CALL dword ptr [eax+12345] 就是 FF 90 00 01 23 45。有趣的是， CALL dword ptr [eax+10] 也可以写成 FF 90 00 00 00 10。至于汇编成哪个二进制形式，这是汇编器的选择。这个类型的指令是6个字节。
</font></p> 

FF 90 00 01 23 45 实际在本机执行的结果是: call qworld ptr [rax + 0x45230100]. 这是因为本机采用了大端存储(big endian), 而原作者应该是小端存储(little endian)。

<p><font color="red">
0xD0 – 0xD7部分则直接是寄存器。这边引用的寄存器的类型有很多，但是在CALL指令当中只能引用通用寄存器，因此 CALL eax 就是 FF D0，臭名昭著的 CALL esp 就是 FF D4。注意 CALL eax 和 CALL [eax] 是不一样的。这些指令也是2个字节。
</font></p>

这里应该叫寄存器寻址。

<p><font color="red">
仔细的人也许主要到了，在表当中，0x14, 0x15, 0x54和0x94是不一样的。0x15比较简单，这个要求 ModR/M后面跟上一个32位立即数作为地址。即常见的 CALL dword ptr [004F778e] 这种格式的，直接跳转到一个固定内存地址处存放的值，常见于调用Windows的导出表。对应的二进制是 FF 15 00 4F 77 8E ，有6个字节。
</font></p>

这里觉得例子也要注意大小端序的问题。

<p><font color="red">
0x14，0x54，0x94部分是最复杂的，因为这个时候，ModR/M不足以指定寻址方式，而是需要一个额外的字节，这个字节就是指令当中的第4个字节，SIB。同样在手册的2.1.3，紧跟着ModR/M的定义：
</font></p>

<p><font color="red">
某些特定的ModR/M字节需要一个后续字节，称为SIB字节。32位指令的基地址+偏移量，以及 比例*偏移量 的形式的寻址方式需要SIB字节。 SIB字节包括下列信息：
</font></p>

-   <font color="red">scale（比例）域指定了放大的比例。</font>
-   <font color="red">index（偏移）域指定了用来存放偏移量 的寄存器。</font>
-  <font color="red"> base （基地址）域用来标识存放基地址的寄存器。</font>

<p><font color="red">
0x14, 0x54, 0x94就是这里所说的“特定的ModR/M字节。这个字节后面跟着的SIB表示了一个复杂的寻址方式，典型的见于虚函数调用：
</font></p>

<p><font color="red">
CALL dword ptr [ecx+4*eax]
</font></p>

<p><font color="red">
就是调用ecx指向的虚表当中的第eax个虚函数。这个指令当中，因为没有立即数，因此FF后面的字节就是0x14，而 [ecx+4*eax] 就需要用SIB字节来表示。在这个指令当中，ecx就是 Base，4是Scale，eax是Index。
</font></p>

注意他说的这个SIB的使用场景。

<p><font color="red">
那么，Base, Scale和Index是如何确定的呢？手册上同样有一张表（又是巨大的表）：

![3332](http://mailshark.nos-jd.163yun.com/document/static/E13BF5C8D1E2E796F9ED47716D6429FD.png)
</font></p>

<p><font color="red">
列是Base，行是Index*Scale，例如[ecx+4*eax] 就是0x81。
</font></p>

ecx是base为2的那一列。4*eax在左边表行有显示, 就对应了0x81。

<p><font color="red">
根据这张表，CALL dword ptr [ecx+4*eax] 就是 FF 14 81 。由此可见，对于 0x14系列的来说，CALL指令就是 3个字节。  
而 0x54 带 8bit 立即数，就是对应于 CALL指令：CALL dword ptr [ecx+4*eax+xx]，这个指令就是 FF 54 81 xx，是4个字节。  
同理，0x94带32位立即数，对应于CALL指令：CALL dword ptr [ecx+4*eax+xxxxxxxx]，这个指令就是 FF 94 81 xx xx xx xx，是7个字节。
</font></p>

注意这个CALL dword ptr [ecx+4*eax+xx]。这里的xx其实是在ModR/M中就明确了， 就是那个54对应的disp8。
同理ecx+4*eax+xxxxxxxx]， 这里ModR/M就是94了。
 
OK，截止到目前，我们基本上能够列出常见的CALL指令的格式了：

|**指令**|**二进制形式**|**实际指令(本机)**|
|---|---|---|
| CALL rel32 | E8 xx xx xx xx | CALL rel32 |
| CALL dword ptr [EAX] | FF 10 | CALL qword ptr [RAX] |
| CALL dword ptr [ECX] | FF 11 | CALL qword ptr [RCX] |
| CALL dword ptr [EDX] | FF 12 | CALL qword ptr [RDX] |
| CALL dword ptr [EBX] | FF 13 | CALL qword ptr [RBX] |
| CALL dword ptr [REG*SCALE+BASE] | FF 14 xx | CALL qword ptr [REG*SCALE+BASE] |
| CALL dword ptr [abs32] | FF 15 xx xx xx xx | CALL qword ptr [abs32] |
| CALL dword ptr [ESI] | FF 16 | CALL qword ptr [RSI] |
| CALL dword ptr [EDI] | FF 17 |  CALL qword ptr [RDI]|
| CALL dword ptr [EAX+xx] | FF 50 xx | CALL qword ptr [RAX+xx] |
| CALL dword ptr [ECX+xx] | FF 51 xx | CALL qword ptr [RCX+xx] |
| CALL dword ptr [EDX+xx] | FF 52 xx | CALL qword ptr [RDX+xx] |
| CALL dword ptr [EBX+xx] | FF 53 xx | CALL qword ptr [RBX+xx] |
| CALL dword ptr [REG*SCALE+BASE+off8] | FF 54 xx xx | CALL qword ptr [REG*SCALE+BASE+off8] |
| CALL dword ptr [EBP+xx] | FF 55 xx | CALL qword ptr [RBP+xx] |
| CALL dword ptr [ESI+xx] | FF 56 xx | CALL qword ptr [RSI+xx] |
| CALL dword ptr [EDI+xx] | FF 57 xx | CALL qword ptr [RDI+xx] |
| CALL dword ptr [EAX+xxxxxxxx] | FF 90 xx xx xx xx | CALL qword ptr [RAX+xxxxxxxx] |
| CALL dword ptr [ECX+xxxxxxxx] | FF 91 xx xx xx xx | CALL qword ptr [RCX+xxxxxxxx] |
| CALL dword ptr [EDX+xxxxxxxx] | FF 92 xx xx xx xx | CALL qword ptr [RDX+xxxxxxxx] |
| CALL dword ptr [EBX+xxxxxxxx] | FF 93 xx xx xx xx | CALL qword ptr [RBX+xxxxxxxx] |
| CALL dword ptr [REG*SCALE+BASE+off32] | FF 94 xx xx xx xx xx | CALL qword ptr [REG*SCALE+BASE+off32] |
| CALL dword ptr [EBP+xxxxxxxx] | FF 95 xx xx xx xx | CALL qword ptr [RBP+xxxxxxxx] |
| CALL dword ptr [ESI+xxxxxxxx] | FF 96 xx xx xx xx | CALL qword ptr [RSI+xxxxxxxx] |
| CALL dword ptr [EDI+xxxxxxxx] | FF 97 xx xx xx xx | CALL qword ptr [RDI+xxxxxxxx] |
| CALL EAX | FF D0 | CALL RAX |
| CALL ECX | FF D1 | CALL RCX |
| CALL EDX | FF D2 | CALL RDX |
| CALL EBX | FF D3 | CALL RBX |
| CALL ESP | FF D4 | CALL RSP |
| CALL EBP | FF D5 | CALL EBP |
| CALL ESI | FF D6 | CALL RSI |
| CALL EDI | FF D7 | CALL RDI |
| CALL FAR seg16:abs32 | 9A xx xx xx xx xx xx |64位不支持,此处不做描述| 

重新翻阅了intel的文档，发现call又有补充了:
FF /3
REX.W + FF /3