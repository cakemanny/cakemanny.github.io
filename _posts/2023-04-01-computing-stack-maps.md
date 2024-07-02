---
layout: post
title: Computing stack maps
date: 2023-04-01
---

You are a rank two compiler writer.
That is, you can write a C compiler
but Go and Scheme are slightly out of reach
because the gory details of how to interface the compiler with the garbage
collector were left as an exercise...
and you prefer couch over exercise.

In this article we will work through some practical problems and their
solutions.

The example will be given in [structlang][1], a simple pedagogical language.
It does nothing useful other than exercise the semantics we are interested in.

[1]: https://github.com/cakemanny/structlang

```
struct Pt { x: int, y: int }
fn f() -> int {
    let a: *Pt = new Pt {1, 2};
    let b: *Pt = new Pt {3, 4};
    a->x + b->y
}
fn g() -> int {
    let c: *Pt = new Pt {5, 6};
    let d: *Pt = new Pt {7, 8};
    c->y + d->x
}
fn main() -> int {
    f() + g()
}
```

It is useful to plan out what the stack frames will look like
during the execution of the program.

<!-- it would be nice to do these side by side moving left to right -->

After entering `main`, the stack contains the return address of the caller of
main (some function in libc), the saved frame pointer of the caller
and the saved value of the callee-saved `x19` register; the latter, so that
`x19` is free to store the result of calling `f` while `g` is being called.
```
                    FP offset
: ...            :
+----------------+
| return addr    |  8
+----------------+
| saved FP       |  0
+----------------+
| spilled x19    | -8
+----------------+
| space to align |  SP
+----------------+
```

Upon entering `f`,
the address to return to (`RA`) in `main` is saved onto the stack
along with the current frame pointer (`FP`)
```
                    FP offset
: ...            :                      : ...            :
+----------------+          ---->       +----------------+           ·+
| return addr    |  8                   | main caller RA |            :
+----------------+                      +----------------+
| saved FP       |  0                   | saved FP       |            main
+----------------+                      +----------------+
| spilled x19    | -8                   | spilled regs   |            :
+----------------+                      | in main ...    |            :
| space to align |  SP                  +----------------+           ·+
+----------------+                      | return addr    |  8         :
:                :                      +----------------+            :
.                .                      | saved FP       |  0
.                .                      +----------------+            f
                                        | space for a    | -8
                                        +----------------+            :
                                        | space for b    | -16        :
                                        +----------------+           ·+
```

The first line in `f`,
`let a: *Pt = new Pt {1, 2};`,
allocates a struct on the heap
and assigns a pointer to it to the variable `a`.
After execution of this,
the space saved for `a` is populated with a heap address.

```
: ...            :          ---->       : ...            :
+----------------+           ·+         +----------------+           ·+
| main caller RA |            :         | main caller RA |            :
+----------------+                      +----------------+
| saved FP       |            main      | saved FP       |            main
+----------------+                      +----------------+
| spilled regs   |            :         | spilled regs   |            :
| in main ...    |            :         | in main ...    |            :
+----------------+           ·+         +----------------+           ·+
| return addr    |  8         :         | return addr    |  8         :
+----------------+            :         +----------------+            :
| saved FP       |  0                   | saved FP       |  0
+----------------+            f         +----------------+            f
| space for a    | -8                  *| a = 0x1234...  | -8
+----------------+            :         +----------------+            :
| space for b    | -16        :         | space for b    | -16        :
+----------------+           ·+         +----------------+           ·+
```

After executing the second line,
`let b: *Pt = new Pt {3, 4};`,
the space reserved for `b` now also contains a heap address.

```
: ...            :          ---->       : ...            :
+----------------+           ·+         +----------------+           ·+
| main caller RA |            :         | main caller RA |            :
+----------------+                      +----------------+
| saved FP       |            main      | saved FP       |            main
+----------------+                      +----------------+
| spilled regs   |            :         | spilled regs   |            :
| in main ...    |            :         | in main ...    |            :
+----------------+           ·+         +----------------+           ·+
| return addr    |  8         :         | return addr    |  8         :
+----------------+            :         +----------------+            :
| saved FP       |  0                   | saved FP       |  0
+----------------+            f         +----------------+            f
| a = 0x1234...  | -8                   | a = 0x1234...  | -8
+----------------+            :         +----------------+            :
| space for b    | -16        :        *| b = 0x5678...  | -16        :
+----------------+           ·+         +----------------+           ·+
```

After returning to main,
the contents of the stack stay the same,
but we have a view shift
i.e. the frame and stack pointers are restored
and they now point at main's frame.

```
: ...            :          ---->       : ...            :
+----------------+           ·+         +----------------+           ·+
| main caller RA |            :         | main caller RA |  8         :
+----------------+                      +----------------+
| saved FP       |            main      | main caller FP |  0         main
+----------------+                      +----------------+
| spilled regs   |            :         | spilled regs   |            :
| in main ...    |            :         | in main ...    | -16 SP     :
+----------------+           ·+         +----------------+           ·+
| return addr    |  8         :         | RA for f call  |
+----------------+            :         +----------------+
| saved FP       |  0                   | FP from f call |
+----------------+            f         +----------------+
| a = 0x1234...  | -8                   | 0x1234...      |
+----------------+            :         +----------------+
| b = 0x5678...  | -16        :         | 0x5678...      |
+----------------+           ·+         +----------------+
```

After entering `g`, space has been reserved for `c` and `d`
but we see that this space is still
populated with values stored there during the execution of `f`,
the addresses that were stored in `a` and `b`.
```
: ...            :          ---->       : ...            :
+----------------+           ·+         +----------------+           ·+
| main caller RA |  8         :         | main caller RA |            :
+----------------+                      +----------------+
| main caller FP |  0         main      | main caller FP |            main
+----------------+                      +----------------+
| spilled regs   |            :         | spilled regs   |            :
| in main ...    | -16 SP     :         | in main ...    |            :
+----------------+           ·+         +----------------+           ·+
| RA for f call  |                     *| return addr    |  8         :
+----------------+                      +----------------+            :
| FP from f call |                     *| saved FP       |  0
+----------------+                      +----------------+            g
| 0x1234...      |                      | c = 0x1234...  | -8
+----------------+                      +----------------+            :
| 0x5678...      |                      | d = 0x5678...  | -16        :
+----------------+                      +----------------+           ·+
```

<!-- This is the important observation.
These are valid addresses in within the heap.  ??
-->
When we come to calculate our stack maps, we must tell the garbage collector
that there are no live pointers here at this time.

After executing the line `let c: *Pt = new Pt {5, 6};` , the space allocated
for `c` contains a live pointer, and the space for `d` still contains the
invalid junk (the old `b`).
```
: ...            :          ---->       : ...            :
+----------------+           ·+         +----------------+           ·+
| main caller RA |            :         | main caller RA |            :
+----------------+                      +----------------+
| main caller FP |            main      | main caller FP |            main
+----------------+                      +----------------+
| spilled regs   |            :         | spilled regs   |            :
| in main ...    |            :         | in main ...    |            :
+----------------+           ·+         +----------------+           ·+
| return addr    |  8         :         | return addr    |  8         :
+----------------+            :         +----------------+            :
| saved FP       |  0                   | saved FP       |  0
+----------------+            g         +----------------+            g
| c = 0x1234...  | -8                  *| c = 0xcccc...  | -8
+----------------+            :         +----------------+            :
| d = 0x5678...  | -16        :         | d = 0x5678...  | -16        :
+----------------+           ·+         +----------------+           ·+
```

And finally after executing `let d: *Pt = new Pt {7, 8};`, the junk
in the spot reserved for `d` is overwritten.

```
: ...            :          ---->       : ...            :
+----------------+           ·+         +----------------+           ·+
| main caller RA |            :         | main caller RA |            :
+----------------+                      +----------------+
| main caller FP |            main      | main caller FP |            main
+----------------+                      +----------------+
| spilled regs   |            :         | spilled regs   |            :
| in main ...    |            :         | in main ...    |            :
+----------------+           ·+         +----------------+           ·+
| return addr    |  8         :         | return addr    |  8         :
+----------------+            :         +----------------+            :
| saved FP       |  0                   | saved FP       |  0
+----------------+            g         +----------------+            g
| c = 0xcccc...  | -8                   | c = 0xcccc...  | -8
+----------------+            :         +----------------+            :
| d = 0x5678...  | -16        :        *| d = 0xdddd...  | -16        :
+----------------+           ·+         +----------------+           ·+
```

The program will complete by dereferencing those heap addresses,
adding together the values from the fields
and then adding that to the result from `f`.
For the sake of this article, however,
that work can all be done in registers and so doesn't affect our stack frames.
_Actually the spilled registers may be relevant
 but we'll come back to them in a future article_.

The main problem we want to focus on today is the one of how to tell the
garbage collector what is junk and what is not.
```
fn g() -> int {
    let c: *Pt = new Pt {5, 6};
    let d: *Pt = new Pt {7, 8};
            //   ^-----------^- what if a GC happens during this allocation
    c->y + d->x
}
```

Our approach? The first step is to keep a record of _defined-variables_ at
each _gc-point_.

_gc-points_ occur at every function call and every use of the `new` operator.
Why is that? `new` allocates memory for a struct, and so may require a
collection to free up some space on the heap, and any function may also
contain uses of `new`.
It may be possible rule out calls to leaf functions that don't use `new`
but let's not worry about that for now.

How do we define _defined-variables_?
Luckily we already have one measure. Variable scoping rules.
Any variable that is in scope at the point of the function
call or `new` usage will be _live_ for the purposes of our garbage
collector.

Let's go through our program and work out the _defined-variables_.

`main` is rather simple.
It has two _gc-points_, the calls to `f` and to `g`,
and no variables in scope at those points,
so quite simple: no _defined-variables_.
```
fn main() -> int {
    f() + g()
//  ^-^   ^-^- gc-point 2
//    \- gc-point 1
}
```


`f` is getting a bit more interesting. We have two _gc-points_,
one when allocating the first `Pt` struct,
the address of which we store in `a`,
and the second when allocating the second `Pt` struct,
address stored in `b`.
```
fn f() -> int {
    let a: *Pt = new Pt {1, 2};
            //   ^-----------^- gc-point 1
    let b: *Pt = new Pt {3, 4};
            //   ^-----------^- gc-point 2
    a->x + b->y
}
```


As we perform semantic analysis of the program we maintain a chain of scopes
while traversing the AST.
We can follow the evolution of our scope chain as we analyse function `f`.


```
                 ->           ->                ->          ->
  upon entering f   gc-point 1      bind a        gc-point 2       bind b
+----------------+             +----------------+            +----------------+
| f, g, main, Pt |             | f, g, main, Pt |            | f, g, main, Pt |
+----------------+             +----------------+            +----------------+
        |                              |                             |
        v                              v                             v
+----------------+             +----------------+            +----------------+
|                |             |       a        |            |    a     b     |
+----------------+             +----------------+            +----------------+
```

We can ignore the root scope which contains all the function, type and global
variable definitions as these will not live on the stack.
At _gc-point_ 1 there are no _defined-variables_ and at _gc-point_ 2 only `a`
has come into scope.


<!-- talk about future AST traversals? -->
Also during semantic analysis,
each variable is given an id to distinguish it from other
variables of the same name but appearing in different scopes.
This has the advantage of allowing later passes not to need to concern
themselves with the scoping rules.

```
+---------+--------+
| symbol  | var_id |
+---------+--------+
| f       |      1 |
| g       |      2 |
| main    |      3 |
| a       |      4 |
| b       |      5 |
| c       |      6 |
| d       |      7 |
+---------+--------+
```

A succinct way to pass along the _defined-variables_ information
might be to use a bitmap
but since we don't know how many variables we'll run into,
we can pass along our _defined-variables_ by attaching an array of these
variable IDs to the AST node.
In some imaginable pretty printing of our AST, it might look like this:

```
Body([
    Let(
        Name=Symbol("a"),
        Type=Ptr(Pointee=Name(Name=Symbol("Pt"))),
        Init=New(Ctor=Symbol("Pt"), Args=[Int(1), Int(2)], DefinedVars=[])
    ),
    Let(
        Name=Symbol("b"),
        Type=Ptr(Pointee=Name(Name=Symbol("Pt"))),
        Init=New(Ctor=Symbol("Pt"), Args=[Int(3), Int(4)], DefinedVars=[4])
                                                                    #   ^
    ),
    Binop(PLUS, Member(Deref(VarRef("a")), "x"), Member(Deref(VarRef("b")), "y"))
])
```


<!--

describe how we built a table of the variables and their
offsets in the stack

Note that this would mostly be eliminated if we did escape analysis and
assigned variable to registers
- caveats - greater than word sized values, escaping variables

and then that we use their pointer disposition (i.e. a pointer map for the
type) to built a general pointer bitmap
and how we can clear the bitmap for slots that are not defined.

we can note that we do this during translation to the IR, so that we don't
need to attach the maps to the AST, but only to the IR - and only to calls

Note a caveat about rewrite stages?
    - rewrite stage temporaries will no longer be live... right? assuming
      they generate no function call and do not allocate

Note how

We can leave them hanging about the details of register spilling.


Parse -> Semantic Analysis -> Frame Layout -> Translation to IR
-> Instruction Selection ->
-> Control Flow Analysis -> Liveness Analysis -> Register Allocation
-> Assembly Emission

end of comment
-->


When we lay out the activation record,
we use the type information to create a pointer map for each variable
in the frame.

```
+-------+------+-----------+----+--------+-------------+
| name  | size | alignment | id | offset | pointer map |
+-------+------+-----------+----+--------+-------------+
| a     |    8 |         8 |  4 |     -8 |  0b00...001 |
+-------+------+-----------+----+--------+-------------+
| b     |    8 |         8 |  5 |    -16 |  0b00...001 |
+-------+------+-----------+----+--------+-------------+
```

In our example here, `a` and `b` are simple single-word variables but it's
also possible to define structs that contain multiple pointers and to store
those on the stack.
For example, `struct Node { value: int, left: *Node, right: *Node }` would
have a pointer map with least significant bits `0b110`.
<!-- Give examples of types with more interesting pointer maps? -->


We delay generation of a frame map for each gc-point until we translate
the AST into the IR (intermediate representation), this saves us needing to
store the _defined-vars_ on each CALL instruction in the IR but also saves
the need for attaching the pointer map to the AST node. (It may be possible
to delay this further, until instruction selection.)

The pointer maps for _defined_ variables are combined together when
translating the call / new expression to create a _locals bitmap_
which describes
local variables area of the frame i.e. which words contain live pointers
when the _gc-point_ is reached, marked with a 1 bit, and which words
contain something else e.g. some integer value or junk from previous
function invocations, marked with a 0 bit.

```
+----------------+                                  ·+
| return addr    |  8                                :
+----------------+                                   :
| saved FP       |  0
+----------------+                                   f
| space for a    | -8 <-------------+-----------+
+----------------+                  |           |    :
| space for b    | -16 <------------|+----------|+   :
+----------------+                  ||          ||  ·+
                                    ||          ||
                                    ||          ||
            locals bitmap:  0b00000000  0b00000010
                            gc-point 1  gc-point 2
```

We've elided it here but there may also be arguments passed on the stack.
In such case, an _arguments bitmap_ for the area north of the frame pointer
will also be created.

<!--

    talk about the saving of the frame maps, with their instruction
-->

During instruction selection we emit a label after each function call.
In the final assembly it looks like the additions shown in the following
snippet.
(The lines prefixed with '+').
`sl_alloc_des` is our allocation function.
`Lret4` labels the `mov` instruction directly after the first call to
`sl_alloc_des` and `Lret5` the `mov` instruction directly after the second.

```diff
 	.globl	_f
 	.p2align	2
 _f:
 	.cfi_startproc
 	stp	x29, x30, [sp, #-16]!
 	mov	fp, sp
 	.cfi_def_cfa w29, 16
 	.cfi_offset w30, -8
 	.cfi_offset w29, -16
 	sub	sp, sp, #16
 L3:
 	adrp	x0, L1@PAGE
 	add	x0, x0, L1@PAGEOFF
 	bl	_sl_alloc_des
+Lret4:
 	mov	w1, #1
 	str	w1, [x0]
 	mov	w1, #2
 	str	w1, [x0, #4]
 	str	x0, [fp, #-8]
 	adrp	x0, L1@PAGE
 	add	x0, x0, L1@PAGEOFF
 	bl	_sl_alloc_des
+Lret5:
 	mov	w1, #3
 	str	w1, [x0]
 	mov	w1, #4
 	str	w1, [x0, #4]
 	str	x0, [fp, #-16]
 	ldr	x0, [fp, #-8]
 	ldr	w1, [x0]
 	ldr	x0, [fp, #-16]
 	ldr	w0, [x0, #4]
 	add	w0, w1, w0
 L2:

 	add	sp, sp, #16
 	ldp	x29, x30, [sp], #16
 	ret
 	.cfi_endproc
```

We store those labels along with the frame maps as fragments that get emitted
after the code in the data segment.

The maps are linked together and given a well-known name for the GC to find.
In the case of structlang, `sl_rt_frame_maps`.
For `f` alone, this could look something like the following snippet of arm64
assembly.
Here, fields related to spilled and callee-saved registers have been omitted.
_As briefly hinted before, I hope to come back to those in a future article._

```
	.section	__DATA,__const
	.p2align	3
Lptrmap0:
	.quad	0	; previous
	.quad	Lret5	; return address - the key
	.short	2	; number of stack args + 2
	.short	2	; length of locals space
	.zero	4
	.quad	0	; arg bitmap
	.quad	2	; locals bitmap     ; 0b00000010 for gc-point 2
	.p2align	3
Lptrmap1:
	.quad	Lptrmap0	; previous
	.quad	Lret4	; return address - the key
	.short	2	; number of stack args + 2
	.short	2	; length of locals space
	.zero	4
	.quad	0	; arg bitmap
	.quad	0	; locals bitmap     ; 0b00000000 for gc-point 1
	.globl	_sl_rt_frame_maps
	.p2align	3
_sl_rt_frame_maps:
	.quad	Lptrmap1
```

