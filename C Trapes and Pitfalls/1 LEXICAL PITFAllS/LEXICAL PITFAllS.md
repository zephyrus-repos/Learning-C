# Lexical Pitfalls

When we read a sentence, we do not usually think about the meaning of the individual  letters of the words that make it up. Indeed, letters mean little by themselves: we group them into words and assign meanings to those words.

So it is also with programs in C and other languages. The individual characters of the program do not mean anything in isolation but only in context. Thus in

```c
p->s "->";
```

the two instances of the `-` character mean two different things. More precisely, each instance of `-` is part of a different **token**: the first is part of `->` and the second is part of a character string. 

> The word **token** refers to a part of a program that plays much the same role as a word in a sentence: in some sense it means the same thing every time it appears. The same sequence of characters can belong to one token in one context and an entirely different token in another context. The part of a compiler that breaks a program up into tokens is often called a *lexical analyzer*.

For another example, consider the statement:

```c
if(x > big) big = x;
```

The first token in this statement is `if`, a keyword. The next token is the left parenthesis, followed by the identifier `x`, the "greater than" symbol, the identifier `big`, and so on. In C, we can always insert extra space (blanks, tabs, or newlines) between tokens.

This chapter will explore some common misunderstandings about the meanings of tokens and the relationship  between tokens and the characters that make them up.

## `=` is not `==`

C use `=` for **assignment** and `==` for **comparison**. This is convenient as assignment is more frequent than comparison.  

This convenience causes a potential problem: one can inadvertently write an assignment where one intended a comparison. Thus, the following statement, which apparently executes a `break` if `x` is equal to `y`:

```c
if (x = y)
    break;
```

actually sets `x` to the value of `y` and then checks whether that value is nonzero.

Consider the following loop, which is intended to skip blanks, tabs, and new lines in a file:

```c
while (c = ' ' || c == '\t' || c == '\n')
    c = getc(f);
```

This loop mistakenly uses `=` instead of `==` in the comparison with `' '`. Because the `=` operator has lower precedence than the `||` operator, the "comparison" actually assigns to `c` the value of the entire expression `' ' || c == '\t' || c == '\n'`. The value of `' '` is nonzero, so this expression evaluates to 1 regardless of the (previous) value of `c`. Thus the loop will eat the entire file. What it does after that depends on whether the particular implementation allows a program to keep reading after it has reached end of file. If it does, the loop will run forever.

Some C compilers try to help their users by giving a warning message for conditions of the form `e1 = e2`. When assigning a value to a variable and' then checking whether the variable is zero, consider making the comparison explicit to avoid warning messages from such compilers. In other words, instead of

```c
if (x = y)
    foo();
```

write:

```c
if((x = y) != 0)
    foo();
```

This will also help make your intentions plain.

It is possible to confuse matters in the other direction too:

```c 
if ((filedesc == open(argv[i], 0) < 0)
	error() ;
```

The `open` function in this example returns `-1` if it detects an error and `0` or a positive number if it succeeds. This fragment is intended to store the result of `open` in `filedesc` and check for success at the same time. However, the first `==` should be `=`. As written, it compares `filedesc` with the result of open and checks whether the result of that comparison is negative. Of course it never is: the result of `==` is always 0 or 1 and never negative. Thus error is not called. Everything appears normal but the value of `filedesc` is whatever it was before, which has nothing to do with the result of `open`.  Some compilers might warn that the comparison with 0 has no effect, but you shouldn't count on it. 

## `&` and `|` are not `&&` and `||`

