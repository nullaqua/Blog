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
变量本质上就是一个字节，我们可以用一个`int`表示变量所在位置，变量编译为后就是若干个>`或`<`操作。
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
if (a) 
{
    ans = 1
}
```
{: file='思路' }
```kotlin
fun not(a: Var, res: Var, cache: Var) = `if`(a, cache, one(res), Code.empty())
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

<div class = "box-info">
未完待续
</div>