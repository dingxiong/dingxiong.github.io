---
layout: post
title: Vim Command Line Area
date: 2025-07-07 00:06 -0700
categories: [editor]
tags: [editor, vim]
---

In Vim, the bottom line where you type commands like `:w`, `:q`, or `/pattern`
is called Command-line area. I use it everyday, but never know its exact syntax
and how the interpreter works. This post is for this purpose.

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

<!-- prettier-ignore-start -->

{% highlight ebnf %}

(* Main Program Structure *)
program ::= command_sequence
command_sequence ::= command_line (newline command_line)*
command_line ::= command ('|' command)*

(* Basic Command Structure *)
command ::= simple_command 
          | control_structure
          | exception_structure
          | loop_structure
          | conditional_structure

simple_command ::= command_name arguments?

(* Control Structures *)
control_structure ::= if_structure
                    | while_structure  
                    | for_structure
                    | try_structure
                    | function_definition

(* Conditional Structures *)
if_structure ::= 'if' condition newline
                 command_sequence
                 ('elseif' condition newline command_sequence)*
                 ('else' newline command_sequence)?
                 'endif'

conditional_structure ::= if_structure

(* Loop Structures *)
while_structure ::= 'while' condition newline
                    command_sequence
                    'endwhile'

for_structure ::= 'for' variable 'in' iterable newline
                  command_sequence
                  'endfor'

loop_structure ::= while_structure | for_structure

(* Exception Handling *)
try_structure ::= 'try' newline
                  command_sequence
                  ('catch' pattern? newline command_sequence)*
                  ('finally' newline command_sequence)?
                  'endtry'

exception_structure ::= try_structure

(* Function Definition *)
function_definition ::= 'function' function_name '(' parameter_list? ')' function_attributes? newline
                       command_sequence
                       'endfunction'

(* Loop Control *)
loop_control ::= 'continue' | 'break'

(* Function Control *)
function_control ::= 'return' expression?

(* File Control *)
file_control ::= 'finish'

(* Terminals and Primitives *)
command_name ::= identifier range_prefix?

range_prefix ::= range ':'?

range ::= line_specifier (',' line_specifier)?

line_specifier ::= number
                 | '.'
                 | '$'
                 | '%'
                 | "'" mark
                 | '/' pattern '/'
                 | '?' pattern '?'
                 | line_specifier ('+' | '-') number?

condition ::= expression

expression ::= (* Complex expression grammar - simplified *)
             | variable
             | literal
             | function_call
             | binary_operation
             | unary_operation

variable ::= identifier

literal ::= string_literal | number_literal

string_literal ::= '"' string_content '"'
                 | "'" string_content "'"

number_literal ::= digit+ ('.' digit+)?

function_call ::= function_name '(' argument_list? ')'

argument_list ::= expression (',' expression)*

parameter_list ::= parameter (',' parameter)*

parameter ::= identifier

function_attributes ::= attribute*

attribute ::= 'range' | 'abort' | 'dict' | 'closure'

binary_operation ::= expression operator expression

unary_operation ::= operator expression

operator ::= '+' | '-' | '*' | '/' | '%' | '==' | '!=' | '<' | '>' | '<=' | '>=' 
           | '&&' | '||' | '=~' | '!~' | 'is' | 'isnot'

arguments ::= argument*

argument ::= expression | flag | option_setting

flag ::= '+' | '-' | '!'

option_setting ::= identifier '=' expression

identifier ::= letter (letter | digit | '_')*

letter ::= 'a'..'z' | 'A'..'Z'

digit ::= '0'..'9'

mark ::= letter

pattern ::= (* Regular expression pattern *)

iterable ::= expression

newline ::= '\n' | '\r\n'

string_content ::= (* Any characters except closing quote *)

function_name ::= identifier

(* Comments *)
comment ::= '"' comment_content

comment_content ::= (* Any characters to end of line *)

{% endhighlight %}

<!-- prettier-ignore-end -->
</details>

It is fairly complicated! You can write `if`, `for` and `while` control flow
statements in the Command-line area. And can also use `|` to pipe multiple
commands into one.

`do_cmdline` sets up the skeleton of the interpreter. The actual implementation
for each semantic keyword/structure is scattered in different places. For
example, `while`'s implementation is
[here](https://github.com/neovim/neovim/blob/8fe4e120a2de558fddbb91ba5438d78f3dbd926a/src/nvim/ex_eval.c#L978).
You can search `void ex_` in file `ex_eval.c`, `ex_cmds.c` and `ex_docmd.c` to
find out what commands Vim has defined and their implementations.

### `:!`

The Bang command is very powerful. The implementation is
[here](https://github.com/neovim/neovim/blob/8fe4e120a2de558fddbb91ba5438d78f3dbd926a/src/nvim/ex_docmd.c#L6451).

There are three mainly use cases.

1. `:!<cmd>`: run cmd and show the output in command-line area.
2. `:r!<cmd>`: run cmd and insert the output to the current cursor location.
   This is actually a compound command. `r` means `read` which is another vim
   command, and pattern `r!` has a special
   [code path](https://github.com/neovim/neovim/blob/8fe4e120a2de558fddbb91ba5438d78f3dbd926a/src/nvim/ex_docmd.c#L5903).
   Therefore, `:r!date` is shorthand of `:read!date`.
3. `%<range>!<cmd>`: use line range as input to the command, and use the output
   to overwrite this range. This is useful for reformatting code.

### `:normal`

`normal` is another commonly used command. For example, how to insert a
character at the end of each line in a range? One way is
`:<range>normal A<char>`.

The implementation is
[here](https://github.com/neovim/neovim/blob/8fe4e120a2de558fddbb91ba5438d78f3dbd926a/src/nvim/ex_docmd.c#L6920).
Basically, it runs the same command for each line in the range selection.
