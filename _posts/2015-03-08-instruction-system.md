---
layout: post
title: "瞬息万变的指令系统"
quote: "计算机通过连续执行每一条的机器语句而实现工作。为了能够更加明了地分析计算机的工作，以下从一个C语言程序的反汇编代码来说明..."
image: false
video: false
---
>此文仅用于MOOC`Linux内核分析`作业

>**张依依**+原创作品转载请注明出处 + 《Linux内核分析》**MOOC课程**http://mooc.study.163.com/course/USTC-1000029000


# 以下为正文


计算机通过连续执行每一条的机器语句而实现工作。主要进行该工作的核心硬件是`CPU`，从某个角度来说，
**CPU**即一个集成电路，实现的功能就是某个位置读出一个指令，从某一个位置读出数据，然后根据指令的不同对数据做不同的处理，然后把结果存回某个地方。



## 反汇编过程
为了能够更加明了地分析计算机的工作，以下从一个C语言程序的反汇编代码来说明：



- 原C程序：

~~~ c

int g(int x)
{
  return x + 7;
}

int f(int x)
{
  return g(x);
}

int main(void)
{
  return f(3) + 2;
}
~~~
- 在终端下的反汇编过程

{% include image.html url="/media/2015-3-8/ter.png" width="100%" description="通过gcc –S –o main.s main.c -m32命令实现" %}

- 反汇编文件

{% include image.html url="/media/2015-3-8/mains.png" width="100%" description="VI中查看" %}

{% include image.html url="/media/2015-3-8/arm.png" width="100%" description="通过把不相关的语句删除后的反汇编程序" %}


## 反汇编程序分析



~~~
g:
	pushl	%ebp
	movl	%esp, %ebp
	movl	8(%ebp), %eax
	addl	$7, %eax
	popl	%ebp
	ret
f:
	pushl	%ebp
	movl	%esp, %ebp
	subl	$4, %esp
	movl	8(%ebp), %eax
	movl	%eax, (%esp)
	call	g
	leave
	ret
main:
	pushl	%ebp
	movl	%esp, %ebp
	subl	$4, %esp
	movl	$3, (%esp)
	call	f
	addl	$2, %eax
	leave
	ret

~~~


* EIP从Main函数的第一句开始
* ` pushl	%ebp`：esp减4，把ebp的值压入栈（即放在标号为1的位置）
* `movl	%esp, %ebp`：这时ebp与esp相同，均指向标号为1的位置
* `subl	$4, %esp`:esp减4，即指向标号为2的位置
* `movl	$3, (%esp)`：标号2的位置放入立即数3
* `call	f`：esp减4,把当前eip指向的位置压入栈（即`call f`的下一条指令），eip指向f函数的`pushl	%ebp`语句


至此堆栈的情况如下：

<table>
  <thead>
    <tr>
      <th>标号</th>
      <th>指向标号的寄存器</th>
      <th>值</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>0</td>
      <td></td>
      <td></td>
    </tr>
    <tr>
      <td>1</td>
      <td>ebp</td>
      <td>ebp->标号0</td>
    </tr>
    <tr>
      <td>2</td>
      <td></td>
      <td>3</td>
    </tr>
    <tr>
      <td>3</td>
      <td>esp</td>
      <td>eip->`addl $2,%eax`语句</td>
    </tr>

  </tbody>
</table>

- `pushl	%ebp`：esp减4，把ebp的值压入栈（即放在标号为4的位置）
- `movl	%esp, %ebp`：这时ebp与esp相同，均指向标号为4的位置
- `subl	$4, %esp`：esp减4，即指向标号为5的位置
- `movl	8(%ebp), %eax`：eax等于ebp指向位置的减8，即标号2的位置
- `movl	%eax, (%esp)`：esp所指的标号5值放入eax的值3
- `call	g`:esp减4,把当前eip指向的位置压入栈（即`call g`的下一条指令），eip指向g函数的`pushl	%ebp`语句
- `pushl	%ebp`：esp减4，把ebp的值压入栈（即放在标号为7的位置）
- `movl	%esp, %ebp`：这时ebp与esp相同，均指向标号为7的位置
- `movl	8(%ebp), %eax`：eax的值等于ebp减8，即标号5中的值3

至此堆栈的情况如下：

<table>
  <thead>
    <tr>
      <th>标号</th>
      <th>指向标号的寄存器</th>
      <th>值（eax->10）</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>0</td>
      <td></td>
      <td></td>
    </tr>
    <tr>
      <td>1</td>
      <td></td>
      <td>ebp->标号0</td>
    </tr>
    <tr>
      <td>2</td>
      <td></td>
      <td>3</td>
    </tr>
    <tr>
      <td>3</td>
      <td></td>
      <td>EIP->`addl $2,%eax`语句</td>
    </tr>
    <tr>
      <td>4</td>
      <td></td>
      <td>ebp->`标号1`</td>
    </tr>
    <tr>
      <td>5</td>
      <td></td>
      <td>3</td>
    </tr>
    <tr>
      <td>6</td>
      <td></td>
      <td>eip->`leave`</td>
    </tr>
    <tr>
      <td>7</td>
      <td>ebp，esp</td>
      <td>ebp->`标号4`</td>
    </tr>
  </tbody>
</table>


* `addl	$7, %eax`：eax中的值加上7,3+7等于10
* `popl	%ebp`：弹出ebp，esp加4，esp指向标号6，ebp指向标号4
* `ret`：弹出eip，eip指向f函数中的`leave`，esp加4，esp指向标号5
* `leave`：esp指向与ebp相同的位置（标号4），弹出ebp，ebp指向标号1，esp加4，指向标号3


至此堆栈的情况如下：

<table>
  <thead>
    <tr>
      <th>标号</th>
      <th>指向标号的寄存器</th>
      <th>值（eax->10）</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>0</td>
      <td></td>
      <td></td>
    </tr>
    <tr>
      <td>1</td>
      <td>ebp</td>
      <td>ebp->标号0</td>
    </tr>
    <tr>
      <td>2</td>
      <td></td>
      <td>3</td>
    </tr>
    <tr>
      <td>3</td>
      <td>esp</td>
      <td>EIP->`addl $2,%eax`语句</td>
    </tr>
  </tbody>
</table>

* `ret`：弹出eip，eip指向主函数的语句`addl $2,%eax`，esp加4，指向标号2
* `addl $2,%eax`：eax加上立即数2,10+2等于12
* `leave`：esp指向与ebp相同的位置（标号1），弹出ebp，ebp指向标号0，esp加4，指向标号1
* `ret`：弹出eip，esp加4，指向标号0



| 最终寄存器的情况  |
|:--------|
| ebp指向标号0    |
| esp指向标号0    |
| eax中最后结果12 ｜


******


## 总结

正如开头所说，计算机是通过连续执行每一条的机器语句而实现工作的。

计算机中执行指令系统主要依靠寄存器以及内存的代码段数据段等的结合，并通过堆栈实现。

站在纯计算机的角度来说，实际上就是一条又一条指令的实现;而站在高级编程语言的角度上，
在函数之间互相调用的过程中，是通过在调用函数前向堆栈中压入当前环境的重要变量实现保护现场的。
这种方式其实与计算机中的程序中断的概念类似。通过esp，ebp，eip这几个寄存器来控制堆栈的情况。
