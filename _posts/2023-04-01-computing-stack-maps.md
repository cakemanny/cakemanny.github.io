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
and the saved value of the callee-saved `x19` register, so that `x19` is free
to store the result of calling `f` while `g` is being called.
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
allocates a struct on the heap and assigns a pointer to it to `a`.

After execution of this, the space saved for `a` is populated with a heap
address.

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

After executing the second line, `let b: *Pt = new Pt {3, 4};`, the space
reserved for `b` now contains also contains a heap address:

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

After returning to main the content of the stack stay the same, we just
have a view shift. i.e. the frame and stack pointers are restored
and now point at main's frame.
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

After entering `g`, we see that the space reserved for `c` and `d` is still
populated with the addresses that were used for `a` and `b`
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

when we come to calculate our stack maps, we must tell the garbarge collector
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

The program will complete by dereferencing those heap addresses and
adding together the values in the fields but for the sake of this article,
that work can be all done in registers and so doesn't affect our stacks.
(Actually, there are some interesting problems there but we'll come back
to them in a future article).

<!--

I guess the plan for this article was to write about recording the defined
(i.e. initialised) variable at each call and new.

-->

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
Why is that? `new` allocates memory for a struct and so may require a
collection to obtain some space on the heap and any function may also contain
the use of `new`.
It may be possible rule out calls to leaf functions that don't use `new` but
let's not worry about that for now.

How do we define _defined-variables_?
Luckily we already have one measure. Variable scoping rules.
Any variable that is in scope at the point of the function
call or `new` usage will be _live_ for the purposes of our garbage
collector.

Let's go through our program and work out the _defined-variables_.

`main` is rather simple. It has two `gc-points`, the calls to `f` and `g`, and
no variables in scope at those points, so quite simple: no _defined-variables_.
```
fn main() -> int {
    f() + g()
}
```
<!-- TODO: come back and annotate in some way -->


`f` is getting a bit more interesting. We have two _gc-points_, one when
allocating the `Pt` to store in `a` and the second when allocating the `Pt`
to store in `b`.
```
fn f() -> int {
    let a: *Pt = new Pt {1, 2};
            //   ^-----------^- gc-point 1
    let b: *Pt = new Pt {3, 4};
            //   ^-----------^- gc-point 2
    a->x + b->y
}
```


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

At _gc-point_ 1 there are no _defined-variables_, at _gc-point_ 2 only `a`
has come into scope.


As we analyse our program each variable is given an id in order to distiguish
variables of the same name without needing to keep the scoping information
around.

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

A succinct way to pass along this information might be to use a bitmap
but since we don't know how many variables we'll run into, we can pass along
our _defined-variables_ by attaching an array of these variable IDs to the
AST node. In some imaginable pretty printing of our AST, this might look like:

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

and then that we use their pointer disposition (i.e. a pointer map for the
type) to built a general pointer bitmap
and how we can clear the bitmap for slots that are not defined.

-->

