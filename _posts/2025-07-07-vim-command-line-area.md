---
layout: post
title: Vim Command Line Area
date: 2025-07-07 00:06 -0700
categories: [editor]
tags: [editor, vim]
---

In Vim, the bottom line where you type commands like `:w`, `:q`, or `/pattern`
is called Command-line area. I used it everyday, but never know the exact
syntax and how the interpreter works.

## Grammar

The top-level function for Command-line interpreter is function
[do_cmdline](https://github.com/neovim/neovim/blob/8fe4e120a2de558fddbb91ba5438d78f3dbd926a/src/nvim/ex_docmd.c#L400).
I copied this whole function to Claude, and Claude told me "The function is
quite sophisticated". Nevertheless, it happily generated an ebnf grammar for
me.

<details>
<summary>
👉 Command-line ebnf grammar 👈
</summary>

{% highlight ebnf %}

(_ Main Program Structure _) program ::= command_sequence command_sequence ::=
command_line (newline command_line)_ command_line ::= command ('|' command)_

(_ Basic Command Structure _) command ::= simple_command | control_structure |
exception_structure | loop_structure | conditional_structure

simple_command ::= command_name arguments?

(_ Control Structures _) control_structure ::= if_structure | while_structure  
 | for_structure | try_structure | function_definition

(_ Conditional Structures _) if_structure ::= 'if' condition newline
command_sequence ('elseif' condition newline command_sequence)\* ('else'
newline command_sequence)? 'endif'

conditional_structure ::= if_structure

(_ Loop Structures _) while_structure ::= 'while' condition newline
command_sequence 'endwhile'

for_structure ::= 'for' variable 'in' iterable newline command_sequence
'endfor'

loop_structure ::= while_structure | for_structure

(_ Exception Handling _) try_structure ::= 'try' newline command_sequence
('catch' pattern? newline command_sequence)\* ('finally' newline
command_sequence)? 'endtry'

exception_structure ::= try_structure

(_ Function Definition _) function_definition ::= 'function' function_name '('
parameter_list? ')' function_attributes? newline command_sequence 'endfunction'

(_ Loop Control _) loop_control ::= 'continue' | 'break'

(_ Function Control _) function_control ::= 'return' expression?

(_ File Control _) file_control ::= 'finish'

(_ Terminals and Primitives _) command_name ::= identifier range_prefix?

range_prefix ::= range ':'?

range ::= line_specifier (',' line_specifier)?

line_specifier ::= number | '.' | '$' | '%' | "'" mark | '/' pattern '/' | '?'
pattern '?' | line_specifier ('+' | '-') number?

condition ::= expression

expression ::= (_ Complex expression grammar - simplified _) | variable |
literal | function_call | binary_operation | unary_operation

variable ::= identifier

literal ::= string_literal | number_literal

string_literal ::= '"' string_content '"' | "'" string_content "'"

number_literal ::= digit+ ('.' digit+)?

function_call ::= function_name '(' argument_list? ')'

argument_list ::= expression (',' expression)\*

parameter_list ::= parameter (',' parameter)\*

parameter ::= identifier

function_attributes ::= attribute\*

attribute ::= 'range' | 'abort' | 'dict' | 'closure'

binary_operation ::= expression operator expression

unary_operation ::= operator expression

operator ::= '+' | '-' | '\*' | '/' | '%' | '==' | '!=' | '<' | '>' | '<=' |
'>=' | '&&' | '||' | '=~' | '!~' | 'is' | 'isnot'

arguments ::= argument\*

argument ::= expression | flag | option_setting

flag ::= '+' | '-' | '!'

option_setting ::= identifier '=' expression

identifier ::= letter (letter | digit | '\_')\*

letter ::= 'a'..'z' | 'A'..'Z'

digit ::= '0'..'9'

mark ::= letter

pattern ::= (_ Regular expression pattern _)

iterable ::= expression

newline ::= '\n' | '\r\n'

string_content ::= (_ Any characters except closing quote _)

function_name ::= identifier

(_ Comments _) comment ::= '"' comment_content

comment_content ::= (_ Any characters to end of line _)

{% endhighlight %}

</details>

It is fairly complicated! You can write `if`, `for` and `while` control flow
statements in the Command-line area. And can also use `|` to pipe multiple
commands into one.

The first question is how vim interpret control flow statement. For example,
`while`'s implementation is
[here](https://github.com/neovim/neovim/blob/8fe4e120a2de558fddbb91ba5438d78f3dbd926a/src/nvim/ex_eval.c#L978).
