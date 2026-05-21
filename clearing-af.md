# A Tiny Z80 Trick: Clearing AF

*By Milos "baze" Bazelides*

On Z80-based systems, stack instructions like `push` are extremely efficient for bulk memory operations. Many games
and demos use this technique to improve throughput, for example when clearing the screen or filling large memory blocks.

There are cases where all register pairs (AF, BC, DE, HL) are needed, and one of them must be zero. Unlike BC, DE, or HL,
the Z80 does not provide a direct instruction such as `ld af,0`. The usual workaround is:
```
      ld    bc,0
      push  bc
      pop   af
```

But there’s a smaller and faster alternative that avoids stack traffic entirely:
```
      xor   a
      ld    b,a
      inc   b
```

After `xor a`, the accumulator is cleared and the F register is set to a known state (zero flag set, all others reset).
After `inc b`, the result is non-zero, and the zero flag is cleared.
