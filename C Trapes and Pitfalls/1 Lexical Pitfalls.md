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

It is easy to miss an inadvertent substitution of `=` for `==` because so many other languages use `=` for comparison. It is also easy to interchange `&` and `&&`, or `|` and `||`, especially because the `&` and `|` operators in C are different from their counterparts in some other languages. 

| operator | meaning                |
| :------: | :--------------------- |
|   `&`    | bit-wise and operation |
|   `&&`   | logical and operation  |
|   `|`    | bit-wise or operation  |
|   `||`   | logical or operation   |

## Greedy lexical analysis

Some C tokens, such as `/`, `*`, and `=`, are only one character long. Other C tokens, such as `/*`, `==`, and identifiers, are several characters long. When a C compiler encounters a `/` followed by an `*`, it must be able to decide whether to treat these two characters as two separate tokens or as one single token. C resolves this question with a simple rule: **repeatedly bite off the biggest possible piece**. That is, the way to convert a C program to tokens is to move from left to right, taking the longest possible token each time. This strategy is also sometimes referred to as *greedy,* or, more colloquially, as the *maximal munch* strategy. Kernighan and Ritchie put it this way: "If the input stream has been parsed into. tokens up to a given character, the next token is taken to include the longest string of characters which could possibly constitute a token." 

Tokens (except string or character constants) never contain embedded white space (blanks, tabs, or newlines). Thus, for instance, `==` is a single token, `= =` is two, and the expression `a---b` means the same as `a-- -b` rather than `a- --b`. 

Similarly, if a `/` is the first character of a token, and the `/` is immediately followed by `*`, the two characters begin a comment, *regardless* of any other context. The following statement looks like it sets `y` to the value of `x` divided by the value pointed to by `p`:

```c
y = x/*p  /* p points at the divisor */;
```

In fact, `/*` begins a comment, so the compiler will simply gobble program text until the `*/` appears. In other words, the statement just sets `y` to the value of `x` and doesn't even look at `p`. Rewriting this statement as

```c
y = x / *p  /* p points at the divisor */;
```

or even

```c
y = x / (*p)  /* p points at the divisor */;
```

would cause it to do the division the comment suggests.

## Integer constants

If the first character of an integer constant is the digit 0, that constant is taken to be in **octal**. Thus `10` and `010` mean very different things. 

Moreover, many C compilers accept 8 and 9 as *"octal"* digits without complaint. The meaning of this strange construct follows from the definition of octal numbers. For instance, `0195` means $1 \times 8^2+ 9 \times 1^1 + 5 \times 8^0$, which is equivalent to 141 (decimal) or 0215 (octal). Obviously we recommend against such usage. ANSI C prohibits it.

Watch out for inadvertent octal values in contexts like this:

```c
#include <stdio.h>

int main(int argc, char *argv[]) {
    struct {
        int part_number;
        char *description;
    } parttab[] = {
        046, "left-handed widget",
        047, "right-handed widget",
        125, "frammis"};

    for (int i = 0; i < 3; i++) {
        printf("%d. part_number: %5d, description: %s\n", i, parttab[i].part_number, parttab[i].description);
    }
    
    return 0;
}
/************ output ************ 
0. part_number:    38, description: left-handed widget
1. part_number:    39, description: right-handed widget
2. part_number:   125, description: frammis
************ output ************/
```

## String and Characters

Single and double quotes mean very different things in C, and confusing them in some contexts will result in surprises rather than error messages.

A character enclosed in single quotes is just another way of writing the integer that corresponds to the given character in the implementation's collating sequence. Thus, in an ASCII implementation, `'a'` means exactly the same thing as `0141` or `97`.

A string enclosed in double quotes, on the other hand, is a short-hand way of writing a pointer to the initial character of a nameless array that has been initialized with the characters between the quotes and an extra character whose binary value is zero.  Thus the statement 

```c
printf("Hello world\n");
```

is equivalent to

```c
char hello[] = {'H', 'e', 'l', 'l', 'o', ' ', 'w', 'o', 'r', 'l', 'd','\n',0};
print(hello);
```

Because a character in single quotes represents an integer and a character in double quotes represents a pointer, compiler type checking will usu ally catch places where one is used for the other. Thus, for example, saying

```c
char *slash = '/';
```

will yield an error message because '/' is not a character pointer. However, some implementations don't check argument types, particularly arguments to `printf`. Thus, saying

```c
printf ('\n'); // ok
```

instead of

```c
printf("\n"); // Incompatible integer to pointer conversion passing 'int' to parameter of type 'const char *'
```

may result in a surprise at run time instead of a compiler diagnostic. 
