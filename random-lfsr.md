# Generating Pseudo-Random Numbers with a Galois Linear Feedback Shift Register

*By Milos "baze" Bazelides*

A Galois Linear Feedback Shift Register (LFSR) is a compact, Z80-friendly pseudo-random number
generator. Its logic maps almost directly to inexpensive CPU opcodes, and its near-uniform
distribution makes it especially useful in demoscene productions.

An $N$-bit LFSR can cycle through up to $2^N − 1$ unique states, provided that the feedback
polynomial (the value `xor`-ed into the register) satisfies certain mathematical properties. For
example, a 16-bit LFSR can generate up to 65535 values before the sequence repeats. The all-zero
state is invalid, since an LFSR initialized with zero will remain stuck at zero forever.

The snippet below generates 8-bit pseudo-random values in the range 1–255, returning the next
value in register A on each call:
```
Lfsr8   ld    a,NON_ZERO_SEED
        add   a,a
        jr    nc,NoXor
        xor   #2D
NoXor   ld    (Lfsr8 + 1),a
```

The 16-bit counterpart stores the pseudo-random value in the HL register pair and executes in
constant time:
```
Lfsr16  ld    hl,NON_ZERO_SEED
        add   hl,hl
        sbc   a,a
        and   #2D
        xor   l
        ld    l,a
        ld    (Lfsr16 + 1),hl
```

Interestingly, the feedback polynomial `#2D` yields maximal-length sequences in both the 8-bit
and 16-bit implementations.
