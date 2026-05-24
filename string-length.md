# Efficient Zero-Terminated String Length

*By Milos "baze" Bazelides*

Occasionally, there is a need to determine the length of a zero-terminated string. To do that, we can use
the `cpir` instruction, which compares the byte at (HL) against A, increments HL, decrements BC, and repeats
until either a match is found or BC reaches zero.

The trick is that if we initialize BC to zero, `cpir` produces the negative string length as a by-product,
which can be negated afterward. If the terminating zero should not be included in the count, it is enough
to omit `scf` from the following code:
```
; Input: HL = string address
; Output: HL = string length

      xor   a
      ld    b,a
      ld    c,a
      cpir
      ld    h,a
      ld    l,a
      scf
      sbc   hl,bc
```

On 8-bit systems, especially when dealing with user input or shorter messages, we can often
assume that strings are at most 255 bytes long, including the terminating zero. Under that
assumption, the routine becomes even more compact:
```
; Input: HL = string address
; Output: A = string length

      xor   a
      ld    c,a
      cpir
      sub   c
      dec   a
```

To exclude the terminating zero from the count, simply omit `dec a`.
