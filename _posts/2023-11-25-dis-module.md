---
title: Python dis module
date: 2023-11-25 23:03 -0800
categories: [programming-language, python]
tags: [dis]
---

# Dis

## Output interpretation

This [stackoverflow post](https://stackoverflow.com/a/47529318/3183330) has a
great diagram: For a sample code

```
from asyncio import sleep
async def f():
    await sleep(0)

```

We have below disassembled bytecode.

```
(1)|(2)|(3)|(4)|          (5)               |(6)|  (7)
---|---|---|---|----------------------------|---|-------
  2|   |   |  0| RETURN_GENERATOR           |   |
   |   |   |  2| POP_TOP                    |   |
   |   |   |  4| RESUME                     | 0 |
  3|   |   |  6| LOAD_GLOBAL                | 1 | (NULL + sleep)
   |   |   | 18| LOAD_CONST                 | 1 | (0)
   |   |   | 20| PRECALL                    | 1 |
   |   |   | 24| CALL                       | 1 |
   |-->|   | 34| GET_AWAITABLE              | 0 |
   |   |   | 36| LOAD_CONST                 | 0 | (None)
   |   |>> | 38| SEND                       | 3 | (to 46)
   |   |   | 40| YIELD_VALUE                |   |
   |   |   | 42| RESUME                     | 3 |
   |   |   | 44| JUMP_BACKWARD_NO_INTERRUPT | 4 | (to 38)
   |   |>> | 46| POP_TOP                    |   |
   |   |   | 48| LOAD_CONST                 | 0 | (None)
   |   |   | 50| RETURN_VALUE               |   |


(1): line number
(2): current instruction executed
(3): possible jump target
(4): bytecode address
(5): opname
(6): argument
(7): human-fridenly argument interpretation
```

As always, I would ask where the code is? Here is
[code](https://github.com/python/cpython/blob/38882d97b35c667ac3d937d82b3d024698e4396a/Lib/dis.py#L292).

Let's talk about column 3 and 7. Column 3 means that this line may have a jump.
And column 7 tells the argument, most likely the jump offset. In the bracket,
it interprets this jump to a bytecode address in the form `(to xxx)`. The
question is how does it know whether a opcode has jump code or not. Here is the
relevant
[code](https://github.com/python/cpython/blob/38882d97b35c667ac3d937d82b3d024698e4396a/Lib/dis.py#L453).
Basically, Cpython maintains two dictionaries of opcodes with potential jumps:
`hasjrel` and `hasjabs`; namely `Has Jump Relative` and `Has Jump Absolute`.

Note, I mean potential jumps. A line like

```
|   |>> | 38| SEND                       | 3 | (to 46)
```

means `SEND` command may jump to address 46. If does not jump, then it will
execute the next opcode. you can read relevant code in `ceval.c` to figure out
when `SEND` jumps.
