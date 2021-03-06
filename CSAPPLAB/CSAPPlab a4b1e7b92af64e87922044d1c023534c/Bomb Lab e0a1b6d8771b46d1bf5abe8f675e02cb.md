# Bomb Lab

前置工作

把bomb机器码反汇编到obj.txt

```bash
objdump -d bomb > obj.txt
```

反汇编函数phase_1

```bash
disas/disassemble phase_1
```

下好断点后，带参数ans.txt运行

```bash
r ans.txt
```

显示addr处的两个字符串

```bash
x/2s addr
p (char*) addr
```

以十六进制显示addr处的20个字节的内存

```bash
x/20x addr
```

显示寄存器内容

```bash
info reg
```

# phase_1

先在obj.txt中静态分析

![Snipaste_2021-12-07_10-24-04.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_10-24-04.png)

可以发现会通过一个比较函数比较地址0x402400处的字符串和输入字符串，不相同就会bomb，相同则函数返回，所以可以动态调试，看一下0x402400处的字符串

对函数phase_1下好断点，带任意一行参数ans.txt运行，然后用gdb的指令查看就行

![Snipaste_2021-12-07_10-27-57.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_10-27-57.png)

会发现一串字符串，我们输入试试看

![Snipaste_2021-12-07_10-29-48.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_10-29-48.png)

第一个炸弹成功拆除

# phase_2

先静态分析，发现需要读入六个数字，如果进到**read_six_numbers**函数中，会发现读入数字小于6会bomb

> 可以根据call、jump指令把基本块分出来，看下call指令之前的mov，参数放哪个寄存器，出来再看看参数变成啥样，继续trace一下
> 

![Snipaste_2021-12-07_10-35-47.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_10-35-47.png)

> 注意x86汇编 mov→ , lea→//?
> 

函数读入数字后会先比较第一个数字是不是1，不是的话就会bomb，所以可以知道第一个数字是1

第一个数字正确之后会跳转到**400f30**处，使rbx指向第二个数字，rbp指向返回地址(rsp+18)，也就是最后一个数字上面的地方，然后jmp到**400f17**处

**400f17**处会把rbx指向的参数的上一个参数的值给到eax，然后把eax翻倍，跟rbx指向的参数进行比较，相当于循环，相等则继续跳转，直到比较完最后一个参数和倒数第二个参数，不相等则bomb

所以从第二个参数开始，每一个参数都应该是前一个参数的两倍

推测答案是**1 2 4 8 16 32**，输入进去试试看，答案正确，炸弹拆除

![Snipaste_2021-12-07_10-44-06.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_10-44-06.png)

# phase_3

先静态看一下，发现有一串字符串跟scanf有关，gdb看一下

![Snipaste_2021-12-07_10-53-05.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_10-53-05.png)

![Snipaste_2021-12-07_10-50-38.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_10-50-38.png)

猜测是输入两个整数

然后第一个整数的值应该小于等于7，不然会bomb，所以第一个整数可以是0 1 2 3 4 5 6 7

像是swith的结构

然后会根据第一个整数的值进行跳转，把跳转的表在gdb里面看一下

![Snipaste_2021-12-07_10-56-39.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_10-56-39.png)

跟汇编代码是对应的

第一个整数是0，跳转到400f7c，eax为207

第一个整数是1，跳转到400fb9，eax为311

第一个整数是2，跳转到400f83，eax为707

第一个整数是3，跳转到400f8a，eax为256

第一个整数是4，跳转到400f91，eax为389

第一个整数是5，跳转到400f98，eax为206

第一个整数是6，跳转到400f9f，eax为682

第一个整数是7，跳转到400fa6，eax为327

因为最后要拿第二个整数跟eax比较，所以可以得出八组答案

```bash
ans:
0 207
1 311
2 707
3 256
4 389
5 206
6 682
7 327
```

随便拿几组试试看，炸弹就拆除了

![Snipaste_2021-12-07_11-03-18.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_11-03-18.png)

# phase_4

```python
	
0x000000000040102e <+34>:    cmp    DWORD PTR [rsp+0x8],0xe		  第一个整数大于0xe就寄
   0x0000000000401033 <+39>:    jbe    0x40103a <phase_4+46>
   0x0000000000401035 <+41>:    call   0x40143a <explode_bomb>
   0x000000000040103a <+46>:    mov    edx,0xe
   0x000000000040103f <+51>:    mov    esi,0x0
   0x0000000000401044 <+56>:    mov    edi,DWORD PTR [rsp+0x8]
   0x0000000000401048 <+60>:    call   0x400fce <func4>				 func4(第一个整数, 0, 0xe);	
   0x000000000040104d <+65>:    test   eax,eax						返回值不为0就寄
   0x000000000040104f <+67>:    jne    0x401058 <phase_4+76>		
   0x0000000000401051 <+69>:    cmp    DWORD PTR [rsp+0xc],0x0		  第二个整数不为0就寄
   0x0000000000401056 <+74>:    je     0x40105d <phase_4+81>
   0x0000000000401058 <+76>:    call   0x40143a <explode_bomb>
```

先静态看一下，发现输入的第一个数应该小于等于0xe，第二个数字应该等于0

![Snipaste_2021-12-07_15-43-56.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_15-43-56.png)

第一个数小于等于0xe后会进入一个func4函数，返回值eax应该等于0，才能不bomb

进func4看一下

![Snipaste_2021-12-07_15-48-12.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_15-48-12.png)

可以看出第一次ecx值为7，然后edi是输入的第一个数字，edi小于等于7而且大于等于7时可以直接跳过递归，返回eax为0，拆除炸弹

所以第一个数字是7第二个数字是0可以拆除炸弹，输入试试看，发现炸弹拆除了

![Snipaste_2021-12-07_15-53-28.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_15-53-28.png)

也可以不跳过递归，进入递归，这样的话就要好好分析一下func4了，可以直接把这个递归函数逆向成伪代码

```c
int func4(int a1, int a2, int x)//a1 = 0xe a2 = 0 x = 第一个数字
{
	int b = (a1 - a2) >> 31;
	int result = ((a1 - a2) + b) >> 1;
	b = result + a2;
	if (b == x)
		return 0;
	else if (b > x)
	{
		result = func4(b - 1, a2, x);
		return result * 2;
	}
	else
	{
		result = func4(a1, b+1, x);
		return result * 2 + 1;
	}
}
```

因为返回值eax要为0，所以可以爆破一下

```c
int main()
{
	int i;
	int a1 = 0xe;
	int a2 = 0;
	for ( i = 0; i <= 0xe; i++)
	{
		if (!func4(a1, a2, i))
		{
			printf("%d\n", i);

		}
	}

	return 0;
}
//0 1 3 7
```

所以答案可以是

```c
0 0
1 0
3 0
7 0
```

# phase_5

先进行静态分析，分析这一部分可以知道程序把输入字符串的地址rdi给了rbx，然后对字符串长度进行判断，不等于6就会bomb

![Snipaste_2021-12-07_18-33-55.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_18-33-55.png)

再来看这一部分，把字符串的第一个字符取出来给ecx，然后让rsp指向它，再把这个字符给rdx，然后异或0xf取低四位，然后再作为0x4024b0地址处字符串的下标，取出值给edx，再让(rsp+10)指向这个取出来的字符，然后循环6次

![Snipaste_2021-12-07_18-37-16.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_18-37-16.png)

先把0x4024b0处地址的字符串打印一下

```bash
pwndbg> x/1s 0x4024b0
0x4024b0 <array.3449>:	"maduiersnfotvbylSo you think you can stop the bomb with ctrl-c, do you?"
```

因为下标是取的低四位，所以范围是0-0xf，所以能取出来的值在**maduiersnfotvby**之中，后面的字符可以不用管

再看这一部分，给(rsp+16)指向的地方放一个0，作为字符串的终止位置，然后把地址0x40245e中的字符串放入esi，然后调用函数进行比较，比较成功就结束，失败就bomb了

![Snipaste_2021-12-07_18-50-15.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_18-50-15.png)

把0x40245e中的字符串拿出来看一下

```bash
pwndbg> x/1s 0x40245e
0x40245e:	"flyers"
```

也就是说我们输入的字符串经过置换，应该等于**"flyers"**

我们把**"flyers"**在数组中的索引都标出来

```bash
f	l	y	e	r	s
9	15	14	5	6	7
```

也就是说输入的字符串取低八位后分别是**9	15	14	5	6	7**

随便找一组答案输入即可，我输入的是**ionefg**

![Snipaste_2021-12-07_18-58-02.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_18-58-02.png)

# phase6

汇编代码太多太乱，分块慢慢看，首先往里读  入六个数字

![Snipaste_2021-12-08_17-01-09.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-08_17-01-09.png)

前面把rsp给了r13，那么地址401117处相当于把rsp中的四位给到eax，也就是输入的第一个数字，然后再把第一个数字减1，跟5比较，小于等于就会跳过炸弹，也就是说第一个数字必须小于等于6

![Snipaste_2021-12-08_17-03-52.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-08_17-03-52.png)

然后跳转到地址401128，r12加1，r12初始值是0，然后跟6比较，等于就跳出，所以这一层大循环的意思应该是验证每一个数字都应该小于等于6

再往下看，地址401132把r12的值给ebx，ebx再给到rax，再把(rsp+rax*4)的值给到eax，相当于rax是rsp的下标，每次取出四个字节，也就是一个我们输入的数字，r12等于1，所以rax等于1，所以相当于取出下一个数字，然后有个 ，把当前数字和下一个数字比较，不相等就跳过炸弹，跳到401145，然后进行下一轮

401145给ebx+1，然后跟0x5进行比较，小于等于就跳转到401135，把ebx给到rax作为下标，跟上面一样，相当于再往后取一个数字，以此类推，这个小循环相当于拿当前数字跟后面的数字作比较，不相等才能跳过炸弹

小循环结束后到40114d这里，这里给r13加0x4，然后进行大循环，大循环就是判断当前数字是否小于等于6，然后进入小循环，判断当前数字跟后面的数字是否相等，以此类推

可以知道这一段代码的作用就是判断每个数字小于等于6且不跟后面的数字相等

结束之后会跳到地址401153，再来看下面这一块汇编

![Snipaste_2021-12-08_17-38-41.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-08_17-38-41.png)

把rsp+0x18给到rsi，rsi实际上指向的是6个数字之后的位置，作为一个标记使用

然后把r14给到rax，r14存的是rsp的位置，相当于把r14给到rax

然后把0x7给到ecx，又把ecx给到edx，然后用edx-0x7存进(rax)中，然后rax+0x4，指向下一个数字

> linux的x86操作向左，数值也存在左边。windows都是向右
> 

![Untitled](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Untitled.png)

然后拿rax跟rsi作比较，看一下有没有指完所有数字，没有的话就继续循环，以此类推

这一块汇编代码相当于把每个数字x的值都变成了7-x

![Snipaste_2021-12-10_11-11-37.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-10_11-11-37.png)

全部变成7-x后进入下一段代码，先给esi置零，然后跳转到401197处，取出一个值给ecx，与1做比较，小于等于1就跳转到401183处，先来看这种情况

401183处，把一个0x6032d0给了edx，可以gdb看一下，发现是个链表

```bash
pwndbg> x/12xg 0x6032d0
0x6032d0 <node1>:	0x000000010000014c	0x00000000006032e0
0x6032e0 <node2>:	0x00000002000000a8	0x00000000006032f0
0x6032f0 <node3>:	0x000000030000039c	0x0000000000603300
0x603300 <node4>:	0x00000004000002b3	0x0000000000603310
0x603310 <node5>:	0x00000005000001dd	0x0000000000603320
0x603320 <node6>:	0x00000006000001bb	0x0000000000000000
```

然后把rdx给到**0x20(%rsp,%rsi,2)**，rsi一开始置零了，所以这里就是给**(rsp+20)**，接下来rsi加4，可以看出一次存放八个字节，然后cmp比较0x18和rsi，说明循环进行6次

接下来又把下一个数的值给ecx，来看看不小于等于1的情况

不小于等于1就会继续往下走，把eax变为1，然后把链表地址给edx，之后跳转到401176

这个地方比较重要，进行是链表的操作，先是进行**mov 0x8(%rdx),%rdx**，相当于把链表指针域的节点给了rdx，相当于rdx变成了下一个节点，node2，然后再给eax+1，然后跟ecx作比较，不相等就接着循环，也就是说会循环**ecx-1**次，也就是链表向后走了**ecx-1**个节点

node向后移动完之后就会跳转到401188处，把向后移动完的node放入以**(rsp+0x20)**为起始位置的栈空间

也就是说之前小于等于1的情况不用循环，直接把node节点放入栈中

总结一下上面的内容，其实就是根据7-x的值，去取移动了ecx-1次的node放入栈中

![Snipaste_2021-12-10_11-47-01.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-10_11-47-01.png)

循环结束后跳转到401188处，把rdx，也就是向后移动了的节点给**0x20(%rsp,%rsi,2)**，6次循环结束后，就跳转到4011ab

然后把**(rsp+20)**的值取出来给rbx，把**(rsp+28)**的地址取出来给rax，把**(rsp+50)**的地址取出来给rsi，

再把rbx给rcx，rcx相当于**(rsp+20)**中的值

之后再把rax中的值取出来给rdx，rdx相当于**(rsp+28)**中的值，然后再把rdx给(rcx+8)，(rcx+8)相当于**(rsp+28)**中的值，

之后对rax加8，然后跟rsi进行比较，rax是**(rsp+0x28)**，rsi是**(rsp+0x50)**，相当于该循环进行5次，该循环作用就是根据链表在栈中的位置重新把链表重新连接，如果循环结束会跳转到4011d2处，把链表节点指向null代表结束了

再来看最后一段代码

![Snipaste_2021-12-10_11-56-32.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-10_11-56-32.png)

该段代码先把0x5给ebp，然后把**(rbx+0x8)**中的值给到rax，rbx的值是之前的**(rsp+0x20)**，rax就相当于是**(rsp+0x28)**，rax就是下一个node的地址了，然后再把该node的数据域的值取出来给eax，然后跟上一个值进行比较，上一个值必须大于下一个值才能跳过炸弹

所以链表重新连接之后应该是前面一个值要大于后面一个值，从大到小排序

```bash
0x6032d0 <node1>:	0x000000010000014c	0x00000000006032e0
0x6032e0 <node2>:	0x00000002000000a8	0x00000000006032f0
0x6032f0 <node3>:	0x000000030000039c	0x0000000000603300
0x603300 <node4>:	0x00000004000002b3	0x0000000000603310
0x603310 <node5>:	0x00000005000001dd	0x0000000000603320
0x603320 <node6>:	0x00000006000001bb	0x0000000000000000
```

先来看看node数据域的大小顺序

node3 > node4 >node5 > node6 > node1 > node2

之前移动node次数是取出来的值-1，但是取出来的值是7-x

要倒退回去

也就是4 3 2 1 6 5

![Snipaste_2021-12-10_12-04-02.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-10_12-04-02.png)

可以看到拆弹成功

# secret_phase

其实把汇编代码往下翻会发现有secret_phase，可以回溯一下，看一下谁调用了它，可以发现每一个phase结束之后都会有一个**phase_defused**，phase_defused中就调用了secret_phase

![Snipaste_2021-12-10_15-41-39.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-10_15-41-39.png)

可以看到三个jne跳转不能成立，不然就会跳过call secre_phase

第一个jne要求603760地址里的数据为6，不过不知道这里面是什么，可以猜测一下是进入了第几个phase这个数字就是几，可以验证一下

分别在phase_5和phase_6下断点，把改地址里的内容打印出来看看

![Snipaste_2021-12-10_15-46-35.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-10_15-46-35.png)

![Snipaste_2021-12-10_15-46-29.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-10_15-46-29.png)

可以看到确实是猜测的这样

然后看一下第二个jne，要求scanf读入三个参数，把相关地址里的值都打印看看

![Snipaste_2021-12-10_15-48-48.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-10_15-48-48.png)

可以猜测输入两个数字和一个字符串，到0x603870的地方，发现这上面已经有了两个数字是我们在phase_4时已经输入了，可以猜测在phase_4后面追加一个符合题目的字符串就可以进去

再看看下一个jne，比较输入的字符串是否相同，把要比较的字符串打印出来看看

![Snipaste_2021-12-10_16-16-31.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-10_16-16-31.png)

所以要在phase_4额外输入一个DrEvil才行

![Snipaste_2021-12-10_16-19-22.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-10_16-19-22.png)

试了一下发现确实进去了

然后去分析一下secret_phase

![Snipaste_2021-12-10_20-59-50.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-10_20-59-50.png)

可以看到先读入一行，然后调用strtol函数，转化为整数，返回的整数保存在rbx中，然后用rax-1去跟0x3e8(1000)作比较，必须小于等于才能跳过炸弹，所以应该输入一个小于等于1001的数字

然后把保存返回整数的ebx给了esi作为call fun7的第二个参数，把0x6030f0作为第一个参数

调用完fun7函数之后，eax如果为0x2，就会跳过炸弹然后成功

然后进入fun7这个关键函数看看

![Snipaste_2021-12-10_21-11-52.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-10_21-11-52.png)

一开始会先检测rdi是否为0，是0的话，直接返回-1，不是想要的2，所以不行

再往下看会把rdi中的值取出给edx，先看看rdi吧

![Snipaste_2021-12-10_21-17-22.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-10_21-17-22.png)

会发现是二叉树，第一个8字节位置存放的是数据，第二个8字节存放的是左子树，第三个8字节存放的是右子树

cmp会去比较当前二叉树的数据与我们输入的值，一共两种情况，一种是当前数据小于等于我们输入的数据，另一种是大于

先来看小于等于的情况

```bash
	401220:	b8 00 00 00 00       	mov    $0x0,%eax          #小于等于
  401225:	39 f2                	cmp    %esi,%edx
  401227:	74 14                	je     40123d <fun7+0x39>
  401229:	48 8b 7f 10          	mov    0x10(%rdi),%rdi   #右子树
  40122d:	e8 d2 ff ff ff       	call   401204 <fun7>
  401232:	8d 44 00 01          	lea    0x1(%rax,%rax,1),%eax
  401236:	eb 05                	jmp    40123d <fun7+0x39>
  401238:	b8 ff ff ff ff       	mov    $0xffffffff,%eax
  40123d:	48 83 c4 08          	add    $0x8,%rsp
  401241:	c3                   	ret
```

当小于等于时会先把eax置0，然后判断输入的值和当前二叉树的数据是否相等，如果相等就直接结束当前函数，不然就让二叉树指向右子树，然后进入fun7函数，出来之后，让返回值rax*2+1，然后结束当前函数

再来看大于的情况，其实差不多

```bash
	401213:	48 8b 7f 08          	mov    0x8(%rdi),%rdi   #左子树 大于
  401217:	e8 e8 ff ff ff       	call   401204 <fun7>
  40121c:	01 c0                	add    %eax,%eax
  40121e:	eb 1d                	jmp    40123d <fun7+0x39>
```

大于时，就让二叉树指向左子树，然后call fun7，ret之后就让eax+eax，然后jmp结束当前函数

我们把二叉树画一下

![Snipaste_2021-12-10_21-28-39.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-10_21-28-39.png)

我们有两种办法让eax等于2

一种是先将eax置0，然后add，然后rax*2+1，然后add，得到eax等于2

另一只是先将eax置0，然后rax*2+1，然后add，得到eax等于2

先来看第一种

只要让二叉树数据先大于我们输入的数据，再小于等于，再大于，再小于等于然后等于退出

就可以得到我们要的指令顺序

二叉树(先左再右再左)，所以我们应该输入0x14(20)

再来看另外一种

只要让二叉树数据先大于我们输入的数据，再小于等于，再小于等于然后等于退出

就可以得到我们要的指令顺序

二叉树(先左再右)，所以我们应该输入0x16(22)

验证一下，成功通过了哈哈

![Snipaste_2021-12-10_21-40-14.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-10_21-40-14.png)

![Snipaste_2021-12-10_21-40-50.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-10_21-40-50.png)