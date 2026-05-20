# A Clever Way to Write 16-bit Loops on the Z80

*By Milos "baze" Bazelides*

Writing loops with an 8-bit counter on the Z80 is straightforward thanks to the dedicated `djnz`
instruction. With 16-bit counters, however, things get a bit more awkward because decrementing
a register pair does not affect the zero flag.

This seemingly odd behavior comes from the Z80’s internal design. The CPU contains a dedicated
16-bit increment/decrement unit designed primarily for updating the Program Counter (PC) and
Stack Pointer (SP) independently of the ALU. That same hardware is also reused for operations
on other register pairs, which is why instructions such as `dec bc` leave the zero flag unaffected.

The traditional way to implement a 16-bit countdown loop looks like this:
```
        ld    bc,COUNT
Loop    ...
        dec   bc
        ld    a,b
        or    c
        jr    nz,Loop
```

Can we do better? As it turns out, yes - at the cost of sacrificing a tiny portion of the full
iteration range:
```
        ld    bc,COUNT + 255
Loop    ...
        dec   bc
        inc   b
        djnz  Loop
```

The trick works because `djnz` decrements the B register and tests it for zero. The `inc b`
instruction compensates for this decrement, except when the correction results in B = 1 before
`djnz`. Consequently, the loop counter must be biased by 255. This is why the valid iteration
range becomes 1..65281 (#FF01) instead of the full 1..65536 range. In practice, this limitation
is rarely a problem. In addition to being both shorter and faster, it also preserves the accumulator.

As far as I know, this neat trick was first discovered by Pavel "Zilog" Cimbal around the turn of
the millennium.
