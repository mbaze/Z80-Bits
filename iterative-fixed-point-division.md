# A New Class of Division Routines on the Zilog Z80: Iterative Fixed-Point Division

*Author: Milos "baze" Bazelides (2026-05-13)*

Most Z80 division routines use some variation of restoring division: a bit-by-bit algorithm that repeatedly shifts,
subtracts, and restores partial remainders. It is reliable and well understood, but it is not always the optimal
choice for certain constant divisors. Instead of constructing the quotient one bit at a time, the approach introduced
here repeatedly refines an approximation using only additions and shifts. The quotient emerges through successive
iteration because it is a stable fixed point of the recursive equation.

To explain the idea, let’s consider division by 3:

$y = x / 3$

We can rearrange this into:

$3 * y = x$

then:

$4 * y = x + y$

and finally:

$y = (x + y) / 4$

That equation is recursive: the value we are trying to compute appears on both sides. This is useful because repeated
iteration drives the estimate toward the unique fixed point $y = x / 3$. If the current estimate is too small, the next
iteration increases it; if it is too large, the next iteration decreases it. On the Z80, division by 4 is simply
a two-bit right shift, so we can repeatedly apply:

`y = (x + y) >> 2`

The useful property is that each iteration divides the remaining error by about 4, so the approximation improves quickly.
The iteration behaves like building the reciprocal through a geometric expansion:

$1 / 3 = 1 / 4 + 1 / 16 + 1 / 64 + ...$

where each pass contributes another term of the series. Even with integer truncation at each step, four iterations are
enough to produce the correct integer quotient over the entire 8-bit range. For 16-bit inputs, eight iterations suffice.

One subtle issue remains: every time we shift right, we lose the fractional part of the value. This introduces a small
downward bias which can be offset by preloading the accumulator with a carefully chosen constant. The constant shifts
the iteration upward so that the truncated process converges to the correct result. For an 8-bit dividend, the preload
constant is floor(256 / 3). For a 16-bit dividend, it is floor(65536 / 3).

The resulting Z80 implementation for division by 3 (A = B / 3) is remarkably compact:
```
      ld    a,256 / 3
      add   a,b
      rra
      srl   a
      add   a,b
      rra
      srl   a
      add   a,b
      rra
      srl   a
      add   a,b
      rra
      srl   a
```

The method generalizes directly to wider operands. Unlike the 8-bit example above, the 16-bit routine below (HL = DE / 3)
is presented in looped form for brevity:
```
      ld    hl,65536 / 3
      ld    b,8
Div3  add   hl,de
      rr    h
      rr    l
      srl   h
      rr    l
      djnz  Div3
```

## Generalizing the idea to divisors of the form 2^N - 1

It turns out that the recursive equation has a particularly elegant form for divisors that are one less than a power of two:
```
 3: y = (x + y) >> 2
 7: y = (x + y) >> 3
15: y = (x + y) >> 4
...
```

For instance, division by 7 (A = B / 7) becomes:
```
      ld    c,%11111110
      ld    a,256 / 7
      add   a,b
      rra
      and   c
      rra
      rra
      add   a,b
      rra
      and   c
      rra
      rra
      add   a,b
      rra
      and   c
      rra
      rra
```

The `%11111110` mask is used because it clears the least significant bit and resets the carry flag, ensuring that
subsequent `rra` instructions behave like chained logical shifts, which would otherwise require slower `srl a` operations.

## Divisors of the form 2^N + 1

Another class of elegant routines emerges for divisors that are one greater than a power of two:
```
 5: y = (x - y) >> 2
 9: y = (x - y) >> 3
17: y = (x - y) >> 4
...
```

This resembles the addition-based recurrence of the form `y = (x + y) >> N`. However, in this case the truncation error
alternates in sign between iterations, so the rounding bias largely cancels itself out. As a result, no preload constant
is required.

For example, division by 5 can be implemented by iterating:

`y = (x - y) >> 2`

which leads to the following routine for A = B / 5:
```
      ld    c,b
      srl   c
      srl   c
      ld    a,b
      sub   c
      rra
      srl   a
      ld    c,a
      ld    a,b
      sub   c
      rra
      srl   a
      ld    c,a
      ld    a,b
      sub   c
      rra
      srl   a
```

The expression $(x - y)$ is harder to implement efficiently because it requires temporary storage. However, it can be
rewritten as `~y + (x + 1)`. The extra increment can be absorbed by pre-incrementing the input value before iteration
begins. Furthermore, because $y$ never exceeds $x$, the bit shifted into the accumulator by `rra` is guaranteed to be
zero. This allows `xor c` in the code below to serve a dual purpose: it both complements the accumulator and clears
the carry flag for subsequent `rra`, producing a tighter inner loop. Although the cycle count is not reduced in this
particular case, it demonstrates a useful contextual optimization:
```
      ld    c,255
      ld    a,b
      and   c
      rra
      rra
      srl   a
      srl   a
      inc   b
      sub   b
      xor   c
      rra
      srl   a
      sub   b
      xor   c
      rra
      srl   a
      sub   b
      xor   c
      rra
      srl   a
```

## Closing Thoughts

These routines are not replacements for general-purpose division. They are specialized iterative solvers for particular
divisors, especially numbers near powers of two. But they represent an interesting and philosophically satisfying
alternative to restoring division: instead of extracting quotient bits one at a time, they converge toward the quotient
using only shifts, additions, and feedback.

For small divisors, the most efficient approach is often a hybrid of direct shifting and these specialized solvers.
Powers of two are handled most efficiently with right shifts, while nearby odd divisors are good candidates for
specialized solvers. Composite divisors can often be reduced to a solver followed by a shift: for example, division by 6
can be implemented as division by 3 followed by a right shift, and division by 10 as division by 5 followed by a right shift.
