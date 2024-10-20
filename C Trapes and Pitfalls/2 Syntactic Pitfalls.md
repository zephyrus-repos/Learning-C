# Syntactic Pitfalls

To understand a C program, it is not enough to understand the tokens that make it up. One must also understand how the tokens combine to form declarations, expressions, statements, and programs. While these combinations are usually well-defined, the definitions are sometimes counter-intuitive or confusing. This chapter looks at some syntactic constructions that are less than obvious.

## Understanding function declarations

Every C variable declaration has two parts: a *type* and a list of expression-like things called *declarators*. A declarator looks something like an expression that is expected to evaluate to the given type. The simplest declarator is a variable:

```c
float f, g;
```

indicates that the expressions `f` and `g`, when evaluated, will be of type float. Because a declarator looks like an expression, parentheses may be used freely:

```c
 float ((f));
```

means that `((f))` evaluates to a float and therefore, by inference, that `f` is also a float. 

Similar logic applies to function and pointer types. For example,

```c
float ff();
```

means that the expression `ff()` is a float, and therefore that `ff` is a function that returns a float. Analogously,

```c
float *pf;
```

means that `*pf` is a float and therefore that `pf` is a pointer to a float.

These forms combine in declarations the same way they do in expressions. Thus

```c
float *g(), (*h)();
```

says that `*g()` and `(*h)()` are float expressions. Since `()` binds more tightly than `*`, `*g()` means the same thing as `*(g())`: `g` is a function that returns a pointer to a float, and `h` is a pointer to a function that returns a float.

Once we know how to declare a variable of a given type, it is easy to write a cast for that type: just remove the variable name and the semicolon from the declaration and enclose the whole thing in parentheses. Thus, since

```c
float (*h)();
```

declares `h` to be a pointer to a function returning a float,

```c
(float (*)());
```

is a cast to a pointer to a function returning a float.

## Operators don't always have the precedence you want

Suppose that the defined constant `FLAG` is an integer with exactly one bit turned on in its binary representation (in other words, a power of two), and you want to test whether the integer variable `flags` has that bit turned on. The usual way to write this is:

```c
if (flags & FLAG) ...
```

The meaning of this is plain to most C programmers: an if statement tests whether the expression in the parentheses evaluates to 0 or not. It might be nice to make this test more explicit for documentation purposes:

```c
if (flags & FLAG != 0) ...
```

The statement is now easier to understand. It is also wrong, because `!=` binds more tightly than `&`, so the interpretation is now:

```c
if (flags & (FLAG != 0)) ...
```

This will work (by coincidence) if `FLAG` is 1 but not otherwise.

Suppose you have two integer variables, `hi` and `low`, whose values are between 0 and 15 inclusive, and you want to set an integer `r` to an 8 bit value whose low-order bits are those of `low` and whose high-order bits are those of `hi`. The natural way to do this is to write:

```c
 r = hi<<4 + low; // wrong
```

Unfortunately, this is wrong. Addition binds more tightly than shifting, so this example is equivalent to:

```c
r = hi << (4 + low);
```

Here are two ways to get it right. The second suggests that the real problem comes from mixing arithmetic and logical operations; the relative precedence of shift and logical operators is more intuitive:

```c
r = (hi << 4) + low;
r = hi << 4 | low;
```

One way to avoid these problems is to parenthesize everything, but expressions with too many parentheses are hard to understand. Thus it may be useful to try to remember the precedence levels in C. The details of pperator precedence is in [C Reference]([C Operator Precedence - cppreference.com](https://en.cppreference.com/w/c/language/operator_precedence)).

## Watch those semicolons!

An extra semicolon in a C program may be harmless: it might be a null statement, which has no effect, or it might elicit a diagnostic message from the compiler, which makes it easy to remove.- One important exception is after an if or while clause, which must be followed by exactly one statement. Consider this example:

```c
if(x[i] > big);
big = x[i];
```

The compiler will happily digest the semicolon on the first line and because of it will treat this program fragment as something quite different from:

```c
if(x[i] > big)
	big = x[i];
```

 The first example is equivalent to:

```c
if(x[i] > big){}
big = x[i];
```

 which is, of course, equivalent to:

```c
big = x[i];
```

Leaving out a semicolon can cause quiet trouble too:

```c
if (n < 3)
	return
logree.date = x[0];
logree.time = x[1];
logree.eode = x[2];
```

Here the return statement is missing a semicolon; yet this fragment may well compile without error, treating the entire statement `logree.date = x[0];` as if it were the operand of the return statement. It is the same as:

```c
if (n < 3)
	return logree.date x[0];
logree.time = x[1];
logree.eode = x[2];
```

If this fragment were part of a function declared to return void, one would expect the compiler to flag it as an error. However, functions that don't return a value are often written with no return type at all, implicitly returning an int. Thus this error may go undetected. Its effect is insidious: if n >= 3, the first of the three assignment statements is simply skipped.

Another place that a semicolon can make a big difference is at the end of a declaration just before a function definition. Consider the following fragment:

```c
struct logrec {
    int data;
    int time;
    int code;
}

main(){
    ...
}
```

There is a semicolon missmg between the first `}` and the definition of `main` that immediately follows it. The effect of this is to declare that the function main returns a struct `logrec`, which is defined as part of this declaration. Think of it this way:

```c
struct logrec {
    int data;
    int time;
    int code;
} main(){
    ...
}
```

If the semicolon were present, `main` would be defined by default as returning an `int`. 

## The switch statement

C is unusual in that the cases in its `switch` statement can flow into each other. Consider, for example, the following program fragments in C:

```c
switch (color) {
    case 1: printf("red");
        break;
    case 2: printf("yellow");
        break;
    case 3: printf("blue");
        break;
}
```

The program fragments do the same thing: print red, yellow, or blue (without starting a new line), depending on whether the variable `color` is 1, 2, or 3. 

Viewing it another way, suppose the C fragment without `break`:

```c 
switch (color) {
    case 1: printf("red");
    case 2: printf("yellow");
    case 3: printf("blue");
}
```

and suppose further that color were' equal to 2. Then the program would print

```shell
yellowblue
```

because control would pass naturally from the second `printf` call to the statement after it.

This is both a strength and a weakness of C `switch` statements. It is a weakness because leaving out a break statement is easy and often gives rise to obscure program misbehavior. It is a strength because by leaving out a break statement deliberately, one can readily express a control structure that is inconvenient to implement otherwise. Specifically, in large switch statements, one often finds that the processing for one of the cases reduces to some other case after relatively little special handling.

For example, consider a program that is an interpreter for some kind of imaginary machine. Such a program might contain a switch statement to handle each of the various operation codes. On such a machine, it is often true that a subtract operation is identical to an add operation after the sign of the second operand has been inverted. Thus, it is nice to be able to write something like this:

```c
case SUBSTRACT:
	opnd2 = -opnd2;
	/* no break */
case ADD:
	...
```

Of course, a comment such as the one in the example above is a good idea; it lets the reader know that the lack of a break statement is intentional.

As another example, consider the part of a compiler that skips white space while looking for a token. Here, one would want to treat spaces, tabs, and newlines identically except that a newline should cause a line counter to be incremented:

```c
case '\n':
	linecount++;
	/* no break */
case '\t':
case ' ':
	...
```

## Calling functions

Unlike some other programming languages, C requires a function call to have an argument list even if there are no arguments. Thus, if `f` is a function,

```c
f();
```

is a statement that calls the function, but

```c
f;
```

does nothing at all. More precisely, it evaluates the address of the function but does not call it.

## The dangling else problem

Although this well-known problem is not unique to C, it has bitten C programmers with many years of experience.

Consider the following program fragment:

```c
if (x == 0)
    if ( y == 0) error();
else {
    z = x + y;
    f(&z);
}
```

The programmer's intention for this fragment is that there should be two main cases:`x=0` and `x!=0`. In the first case, the fragment should do nothing at all unless `y=0`, in which case it should call `error()`. In the second case, the' program should set `z` to `x+y` and then call `f` with the address of `z` as its argument.

However, the program fragment actually does something quite different. The reason is the rule that an else is always associated with the closest unmatched if inside the same pair of braces. if we were to indent this fragment the way it is actually executed, it would look like this:

```c
if (x == 0) {
    if ( y == 0) error();
	else {
    	z = x + y;
    	f(&z);
	}
}
```

In other words, nothing at all will happen if `x!=O`. To get the effect implied by the indentation of the original example, write:

```c
if (x == 0) {
    if ( y == 0) error();
}
else {
    z = x + y;
    f(&z);
}
```

The `else` here is associated with the first `if`, even though the second `if` is closer, because the second one is now enclosed in braces.



