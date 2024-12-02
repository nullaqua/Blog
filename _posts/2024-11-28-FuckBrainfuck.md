---
title: FuckBrainfuck
date: 2024-11-28 22:00:00 +0800
tags:
  - develop
  - kotlin
author: NullAqua
categories:
  - develop
  - other
math: true
---

# FuckBrainfuck
<div class="box-warning">
<div class="title"> 先叠甲 </div>
以下内容纯属娱乐，不具有任何实际意义。我无法保证我的实现没有问题，如有问题欢迎指出。
</div>
## Brainfuck简述
若你已经知道何为Brainfuck,请跳过此部分.
Brainfuck是一种极简单的编程语言,由Urban Müller于1993年创造。它只包含八个命令，分别是`>` `<` `+` `-` `[` `]` `.` `,`。它的内存模型是一个无限长度的字节数组,每个字节的初始值为$0$。指针指向当前字节.每个命令的含义如下：
- `>`：指针向右移动一个位置
- `<`：指针向左移动一个位置
- `+`：当前字节值加一
- `-`：当前字节值减一
- `[`：若当前字节值为$0$，则跳转到对应`]`后面的指令
- `]`：若当前字节值不为$0$，则跳转到对应`[`后面的指令
- `.`：输出当前字节的ASCII码对应的字符
- `,`：读入一个字符，存入当前字节
虽然该语言极其简单，但其图灵完备性已经被证明。并且由于其极简的语法，Brainfuck程序通常非常难以阅读和编写。因而得名Brainfuck.

## FuckBrainfuck
因Brainfuck图灵完备，因此理论上我们可以设计一个高级语言，使之能够编译成Brainfuck程序。这样我们就可以用高级语言编写Brainfuck程序，而不必直接用Brainfuck语言。这就是FuckBrainfuck的目的。
### FuckBrainfuck语法
使用`antlr`定义语法如下：

```kotlin
grammar FuckBrainfuck;  
  
program: (functionDeclaration)* EOF;  
  
functionDeclaration  
    : inlineFunctionModifier? 'fun' IDENT '(' parameterList? ')' (':' (type | 'void'))? block  
    ;  
  
inlineFunctionModifier  
    : 'inline'  
    ;  
parameterList  
    : parameter (',' parameter)*  
    ;  
parameter  
    : referenceModifier? IDENT ':' type  
    ;  
referenceModifier  
    : '&'  
    ;  
statementList  
    : statement* // 可以根据需要定义具体的语句类型  
    ;  
statement  
    : variableDeclaration  
    | variableAssignment  
    | putStatement  
    | returnStatement  
    | block  
    | ifStatement  
    | whileStatement  
    | expressionStatement  
    ;  
variableDeclaration  
    : 'var' referenceModifier? IDENT ':' type ( '=' expression )? ';'  
    ;  
variableAssignment  
    : IDENT '=' expression ';'  
    ;  
putStatement  
    : 'put' expression ';'  
    ;  
returnStatement  
    : 'return' expression? ';'  
    ;  
block  
    : '{' statementList '}'  
    ;  
ifStatement  
    : 'if' '(' expression ')' statement ('else' statement)?  
    ;  
whileStatement  
    : 'while' '(' expression ')' statement  
    ;  
expressionStatement  
    : expression ';'  
    ;  
  
type  
    : 'unit'  
    ;  
  
expression  
    : level2Expression (ADD level2Expression)*  
    | IDENT  
    ;  
ADD: '+' | '-';  
  
level2Expression  
    : level1Expression (MUL level1Expression)*  
    ;  
MUL: '*' | '/';  
  
level1Expression  
    : '(' expression ')'  
    | functionCall  
    | IDENT  
    | NUMBER  
    | STRING  
    | CHAR  
    ;  
  
functionCall  
    : IDENT '(' argumentList? ')'  
    ;  
argumentList  
    : expression (',' expression)*  
    ;  
  
NUMBER: [0-9]+; // 数字规则  
STRING: '"' (~["])* '"'; // 字符串规则  
CHAR: '\'' (~[']) '\''; // 字符规则  
IDENT: [a-zA-Z_][a-zA-Z0-9_]*; // 标识符规则  
WS: [ \t\r\n]+ -> skip; // 跳过空白字符
```
{: file='fuckBrainfuck.g4' }

### 基本实现思路
每次执行一个操作后，令指针回归原位。这样每个操作间就可以做到完全独立。因此我们可以将每个操作编译成一段Brainfuck代码，然后将这些代码连接起来，就得到了一个Brainfuck程序。
### 基本组件的实现
#### var
变量本质上就是一个字节，我们可以用一个`int`表示变量所在位置，变量编译为后就是若干个`>`或`<`操作。
```kotlin
@JvmInline  
@Suppress("NOTHING_TO_INLINE")  
value class Var(val value: Int)  
{  
    inline operator fun plus(b: Var) = Var(value + b.value)  
    inline operator fun plus(b: Int) = Var(value + b)  
    inline operator fun minus(b: Var) = Var(value - b.value)  
    inline operator fun minus(b: Int) = Var(value - b)  
    inline operator fun compareTo(b: Var) = value.compareTo(b.value)  
    inline operator fun plus(c: Code) = Code(toString() + c.value)  
    inline operator fun inc() = Var(value + 1)  
    inline operator fun dec() = Var(value - 1)  
    inline operator fun unaryMinus() = Var(-value)  
    inline operator fun unaryPlus() = this  
    override fun toString() = if (value <= 0) "<".repeat(abs(value)) else ">".repeat(abs(value))  
    companion object  
    {  
        inline fun now() = Var(0)  
    }  
}
```
#### CodeBlock
这纯粹是为方便编程而设计的，可以直接将代码段当作字符串拼接。
```kotlin
@JvmInline  
@Suppress("NOTHING_TO_INLINE")  
value class Code(val value: String)  
{  
    inline operator fun plus(b: Code) = Code(value + b.value)  
    inline operator fun plus(b: String) = Code(value + b)  
    inline operator fun plus(b: Var) = Code(value + b)  
    override fun toString() = value  
    companion object  
    {  
        inline fun String.asCode() = Code(this)  
        inline fun Char.asCode() = Code(this.toString())  
        inline fun Var.asCode() = Code(this.toString())  
        inline fun empty() = "".asCode()  
    }  
}
```
#### runOn
前面已经提到了，每个操作是独立的，每次移动到目标位置，执行完操作后，需要回到原位。这个函数就是用来实现这个功能的。
```kotlin
fun runOn(a: Var, block: Code): Code = a + block + (-a)
```
#### inc/dec
```kotlin
fun inc(a: Var, value: UByte = 1u): Code = runOn(a, "+".repeat(value.toInt()).asCode())
fun dec(a: Var, value: UByte = 1u): Code = runOn(a, "-".repeat(value.toInt()).asCode())
```
#### put/get
```kotlin
fun put(a: Var): Code = runOn(a, ".".asCode())
fun get(a: Var): Code = runOn(a, ",".asCode())
```
#### while
```kotlin
fun `while`(a: Var, block: Code): Code = runOn(a, '['.asCode()) + block + runOn(a, ']'.asCode())  
fun `while`(a: Var, block: () -> Code): Code = `while`(a, block())
```
#### setValue
通过`[-]`可以将当前位置的值设置为$0$。之后配合自增操作，可以将当前位置的值设置为任意值。
（这里实现的区间赋值）
```kotlin
fun setValue(a: Var, value: Int, size: Int = 1): Code  
{  
    var code = Code.empty()  
    for (i in 0 until size)  
        code += runOn(a + Var(i), ("[-]" + "+".repeat(value.toInt())).asCode())  
    return code  
}  
fun zero(a: Var, size: Int = 1): Code = setValue(a, 0, size)  
fun one(a: Var, size: Int = 1): Code = setValue(a, 1, size)
```
#### move
用已实现的代码进行组合，`while(a) { --a; ++b; }`
```kotlin
fun move(a: Var, b: Var): Code =  
    if (a == b) Code.empty() else `while` (a, inc(b) + dec(a))  
fun move(a: Var, b: Var, size: Int): Code  
{  
    var code = Code.empty()  
    for (i in 0 until size)  
        code += move(a + Var(i), b + Var(i))  
    return code  
}
```
#### if
逻辑是这样的：
```kotlin
if (a) { trueBlock } else { falseBlock }
----------
var cache = 1;
while (a)
{
    a = 0
    cache = 0
    trueBlock
}
while (cache)
{
    cache = 0;
    falseBlock;
}
```
其最大问题是在判断后，需要将`a`的值还原，但为判断`a`的值就不得不出现`while(a)`而为退出`while`循环就不得不出现`a=0`。这样就会导致`a`的值被修改。因此这个实现并不完美。但我目前没有找到更好的方法。
```kotlin
fun `if`(a: Var, cache: Var, trueBlock: Code, falseBlock: Code): Code =  
    one(cache) +  
    runOn(a, '['.asCode()) +  
    zero(a) +  
    dec(cache) +  
    trueBlock +  
    runOn(a, ']'.asCode()) +  
    runOn(cache, '['.asCode()) +  
    dec(cache) +  
    falseBlock +  
    runOn(cache, ']'.asCode())  
fun `if`(a: Var, cache: Var, trueBlock: () -> Code, falseBlock: () -> Code): Code =  
    `if`(a, cache, trueBlock(), falseBlock())
fun ifWithCopy(a: Var, cache1: Var, cache2: Var, trueBlock: Code, falseBlock: Code): Code =  
    copy(a, cache1, cache2) +  
    `if`(cache1, cache2, trueBlock, falseBlock)
```
#### copy
大概思路是这样的：
```kotlin
var cache = 0
while (a)
{
    ++b
    ++cache
}
move(cache, a)
```
也就是先将`a`的值同时移动到`b`和`cache`，然后再将`cache`的值移动回`a`。
```kotlin
fun copy(a: Var, b: Var, cache: Var): Code =  
    `while`(a, inc(b) + inc(cache) + dec(a)) +  
    move(cache, a)  
  
fun copy(a: Var, b: Var, cache: Var, size: Int): Code =  
    (0..<size).map { copy(a + Var(it), b + Var(it), cache) }.reduce(Code::plus)
```
#### not
```kotlin
if (a) {}
else 
{
    ++ans
}
```
{: file='思路' }
```kotlin
fun not(a: Var, res: Var, cache: Var) = `if`(a, cache, Code.empty(), inc(res))
```
{: file='实现' }
#### and
```kotlin
if (a)
{
    if (b)
    {
        ans = 1
    }
}
```
{: file='思路' }
```kotlin
fun and(a: Var, b: Var, ans: Var, cache: Var): Code =  
    `if`(a, cache, `if`(b, cache, one(ans), Code.empty()), Code.empty())
```
{: file='实现' }
#### or
```kotlin
if (a)
{
    ans = 1
}
else if (b)
{
    ans = 1
}
```
{: file='思路' }
```kotlin
fun or(a: Var, b: Var, ans: Var, cache: Var) =  
    `if`(a, cache, one(ans), `if`(b, cache, one(ans), Code.empty()))
```
{: file='实现' }
#### xor 
```kotlin
if (a)
{
    if (b) {}
    else
    {
        ans = 1
    }
}
else
{
    if (b) 
    {
        ans = 1
    }
}
```
{: file='思路' }
```kotlin
fun xor(a: Var, b: Var, ans: Var, cache: Var): Code =  
    `if`(a, cache,  
        `if`(b, cache, Code.empty(), one(ans)),  
        `if`(b, cache, inc(ans), Code.empty())  
    )
```
{: file='实现' }
#### add/sub
上面已经实现了`move`，观察发现，其实`move`就相当于实现了`b+=a`，只不过会清空`a`。因此`a+=b`就是`move(b, a)`。
而`a-=b`只需要把`move`中的`inc`改为`dec`即可。
```kotlin
fun addAssign(a: Var, b: Var): Code = move(b, a)  
fun subAssign(a: Var, b: Var): Code =  
    if (a == b) Code.empty() else `while` (b, dec(a) + dec(b))  
// 加减常数
fun addAssign(a: Var, value: UByte): Code =  
    runOn(a, "+".repeat(value.toInt()).asCode())  
fun subAssign(a: Var, value: UByte): Code =  
    runOn(a, "-".repeat(value.toInt()).asCode())
```
#### equal/notEqual
其实就是`a-b`，然后判断`a`是否为$0$。若为$0$则`ans`为$1$，否则为$0$。
因此这里不赘述了。
#### less/greater
将`b`逐渐减为$0$，期间若出现`a`为$0$，则说明`a<b`
```kotlin
var ans = 0
var cache = 0
while (b)
{
    --b
    if (a) --a
    else
    {
        ++ans
        b = 0
    }
}
```
{: file='思路' }
```kotlin
fun less(a: Var, b: Var, ans: Var, cache1: Var, cache2: Var): Code =  
    `while`(b)  
    {  
        dec(b) +  
        ifWithCopy(  
            a,  
            cache1,  
            cache2,  
            dec(a),  
            inc(ans) +  
            zero(b),  
        )  
    } +  
    zero(a)  
  
fun greater(a: Var, b: Var, ans: Var, cache1: Var, cache2: Var): Code =  
    less(b, a, ans, cache1, cache2)
```
{: file='实现' }
#### lessEqual/greaterEqual
只需要在上面的基础上在`while`后判断若`a`为$0$则说明`a==b`答案也为$1$
```kotlin
var ans = 0
var cache = 0
while (b)
{
    --b
    cache = a
    // 因为if会导致b被清空，所以先复制
    if (cache) --a
    else
    {
        ++ans
        b = 0
    }
}
// if的同时a也被清空，不需要再次清空
if (a) {}
else ++ans
```
{: file = '思路' }
```kotlin
fun lessEqual(a: Var, b: Var, ans: Var, cache1: Var, cache2: Var): Code =  
    `while`(b)  
    {  
        dec(b) +  
        copy(a, cache1, cache2) +  
        `if`(  
            cache1,  
            cache2,  
            dec(a),  
            inc(ans) +  
            zero(b),  
        )  
    } +  
    `if`(a, cache1, Code.empty(), inc(ans))  
  
fun greaterEqual(a: Var, b: Var, ans: Var, cache1: Var, cache2: Var): Code =  
    lessEqual(b, a, ans, cache1, cache2)
```
{: file='实现' }
#### mul
```kotlin
var cache = 0
move(a, cache)
while (cache)
{
    --cache
    copy(b, a)
}
```
{: file='思路' }
```kotlin
fun mulAssign(a: Var, b: Var, cache: Var, cache2: Var): Code =  
    move(a, cache) +  
    `while`(cache)  
    {  
        copy(b, a, cache2) +  
        dec(cache)  
    } +  
    zero(b) +  
    zero(cache2)
```
{: file='实现' }
#### div
不停的给`a`减去`b`，并给答案加一，直到某次在减的过程中`a`为$0$了，结束。
内层的`while`循环相当于`a-=b`的同时进行了判断了`cahce`是否大于`b`，比较结果存在了`cahce3`中。
```kotlin
var cache = 0
move(a, cache)
while (cache)
{
    var cache1 = b
    while (cache1)
    {
        if (cache) {}
        else ++cache3
        --cache1
        --cache
    }
    if (cache3) cache = 0
    else ++a
}
```
{: file='思路' }
```kotlin
fun divAssign(a: Var, b: Var, cache: Var): Code =  
    move(a, cache) +  
    `while`(cache)  
    {  
        copy(b, cache + 1, cache + 2) +  
        `while`(cache + 1)  
        {  
            ifWithCopy(cache, cache + 4, cache + 2, Code.empty(), inc(cache + 3) + zero(cache + 1)) +  
            dec(cache + 1) +  
            dec(cache)  
        } +  
        `if`(cache + 3, cache + 2, zero(cache), inc(a))  
    } +  
    zero(b) +  
    zero(cache, 5)
```
{: file='实现' }

> 因为所需`cache`过多，所以传一个基准位置，相当于占用了基准位置开始的5个字节。
{: .prompt-info }
### 函数调用及调用栈
#### 实现难点
##### 变量位置的不确定
上面我们基础组件实现的实现前提是变量的位置在编译时已知，如果不考虑函数调用，那么这是可行的。但如果考虑考虑函数调用，那么变量的位置就会变得非常复杂（编译时不确定其实就是涉及到运行时指针解引用，其实可以做到，但是很麻烦，而且效率低下）。
##### 当前应执行哪个代码块
函数调用意味着需要记录当前执行到哪个函数的哪个位置，当调用其他函数时，需要记录当前函数的位置，等到调用结束后，需要恢复当前函数的位置。
#### 实现思路
##### 变量位置的不确定
虽然由于变量的绝对位置并不确定，但是每个变量在函数内部的相对位置是确定的。因此我们可以这样解决函数调用时变量位置的问题：
```kotlin
fun a()
{
    var a: unit;
    var b: unit;
    c();
    var c: unit;
}
fun b()
{
    var a: unit;
    c();
    var b: unit;
}
fun c()
{
    var a: unit = '1';
    var b: unit = a;
    put b;
}
```
{: file='示例' }
当调用`c`时，`a`和`b`的位置相对于`c`是确定的，但`c`到底在哪里是不确定的。也就是说，整个`c`函数的位置整体偏移了一个不确定的距离。而这个距离显然和调用`c`的函数有关。因此我们可以在`c`中不考虑整体的偏移，在`a`、`b`函数执行调用的时候先偏移一定的距离再执行`c`的逻辑。
例如上面的代码可以编译为：
```kotlin
fun a()
{
    var a: unit; // 编译为bf没有实际行为
    var b: unit; // 编译为bf没有实际行为
    c(); // 此时c应当整体偏移2个字节，所以我们可以编译为：>>(执行c)<<
    var c: unit; // 编译为bf没有实际行为
}
fun b()
{
    var a: unit; // 编译为bf没有实际行为
    c(); // 此时c应当整体偏移1个字节，所以我们可以编译为：>(执行c)<
    var b: unit; // 编译为bf没有实际行为
}
fun c()
{
    var a: unit = '1'; // 编译为bf：[-]+++++一堆加号加到98
    var b: unit = a; // 一个复制，见上面实现的基础组件
    put b; // >.< 移动到b并打印
}
```
也就是提前位移，然后执行函数的逻辑。这样也可以实现函数的递归调用。
例如：
```kotlin
fun a()
{
    var a: unit; // 编译为bf没有实际行为
    var b: unit; // 编译为bf没有实际行为
    a(); // 此时a应当整体偏移2个字节，所以我们可以编译为：>>(执行a)<<
    var c: unit; // 编译为bf没有实际行为
}
```
每次新的`a`在栈中都会相对于当前的`a`偏移$2$个字节。这样就可以实现递归调用。
##### 当前应执行哪个代码块
上面我们已经实现了维护变量位置的方法，但是如何维护当前应执行哪个代码块呢？其实很简单，在每个函数的栈中，保留一个字节用于记录当前执行的位置，一个字节记录调用当前函数的函数用于退栈即可。然后将所有代码全部包裹在一个巨大的`while`中。
还是以上面的为例，我们可以编译为：
```kotlin
while (/*当前位置*/) // 当前位置应是当前函数保留的用于记录正在执行的代码的变量
{
    if (/*当前位置*/ == 1) // 1代表a函数调用c之前的代码
    {
        // 声明a变量，没有实际代码
        // 声明b变量，没有实际代码
        
        // 调用c
        >> // 按上面的规则，c函数应当偏移2个字节
        [-]++ // 然后把第一个保留字节设为2，表示c函数退栈后应执行2的代码，即a函数后半段代码
        >[-]+++++ // 然后把第二个保留字节设为5，表示要执行c函数
        // 最终指针位于第二个保留字节，即当前字节为5，因此外层循环将执行if (..==5)的部分，即c
    }
    if (/*当前位置*/ == 2) // 2代表a函数调用c之后的代码
    {
        // 此时第一件事是实现c的退栈，当前指针应指在c的第一个保留字节上，这个字节的值应当是2
        << // 然后回到a函数的位置
        // 声明c变量，没有实际代码
        < // a函数执行结束，退到第一个保留字节的位置，执行退栈
    }
    if (/*当前位置*/ == 3) // 3代表b函数调用c之前的代码
    {
        // 声明a变量，没有实际代码
        
        // 调用c
        > // 按上面的规则，c函数应当偏移1个字节
        [-]++++ // 然后把第一个保留字节设为2，表示c函数退栈后应执行2的代码，即b函数后半段代码
        >[-]+++ // 然后把第二个保留字节设为3，表示要执行c函数
    }
    if (/*当前位置*/ == 4) // 4代表b函数调用c之后的代码
    {
        // 此时第一件事是实现c的退栈，当前指针应指在c的第一个保留字节上，这个字节的值应当是4
        << // 然后回到b函数的位置
        // 声明b变量，没有实际代码
        < // b函数执行结束，退到第一个保留字节的位置，执行退栈
    }
    if (/*当前位置*/ == 5) // 5代表c函数
    {
        // 声明a变量，当前位置是保留字节，所以a的位置应当是当前位置+1
        >[-]+++++++++<// 48个+，省略了
        // 声明b变量，当前位置是保留字节，所以b的位置应当是当前位置+2
        copy(a,b) // 见上面的copy组件
        // 输出b
        >>.
        // 退栈
        < // 退到第一个保留字节的位置
        // end
    }
}
```
也就是说，指针所指的字节永远是用于表示当前要执行哪段代码的。每次调用其他函数时，在目标位置写入2个保留字节用于指示当期执行哪个函数以及退栈时应执行哪段代码。每次函数退栈时，指针只需左移一个，回到第一个保留字节，然后根据这个字节的值执行对应的退栈操作即可。
显然，用这种方法也可以实现`goto`，只需要将移动到c并写入c的保留字节的部分改为修改当前的保留字节即可。
> 别急还没写完呢^v^
{: .prompt-warning }
