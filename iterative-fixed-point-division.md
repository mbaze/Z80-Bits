# A New Class of Division Routines on the Zilog Z80: Iterative Fixed-Point Division

Author: Milos "baze" Bazelides (2026-05-13)

Most Z80 division routines use some variation of restoring division: a bit-by-bit algorithm that repeatedly
shifts, subtracts, and restores partial remainders. It is reliable and well understood, but it is not always
the most efficient choice for certain constant divisors. Instead of constructing the quotient one bit at a time,
the approach introduced here repeatedly refines an approximation using only additions and shifts. The quotient
emerges through successive iteration because it is a stable fixed point of the recursive equation.

To explain the idea, let’s consider division by 3:

y = x / 3

We can rearrange this into:

3 * y = x

then:

4 * y = x + y

and finally:

y = (x + y) / 4

That equation is recursive: the value we are trying to compute appears on both sides. This is useful because
repeated iteration drives the estimate toward the unique fixed point y = x / 3. If the current estimate is too small,
the next iteration becomes larger; if it is too large, the next iteration becomes smaller. On the Z80, division
by 4 is simply a two-bit right shift, so we can repeatedly apply:

y = (x + y) >> 2

The remarkable part is that the error shrinks very quickly. Every iteration divides the remaining error by 4,
so each pass contributes roughly two additional bits of precision to the quotient. In effect, the iteration
computes the reciprocal through a geometric expansion:

1 / 3 = 1 / 4 + 1 / 16 + 1 / 64 + ...

where each iteration contributes another term of the expansion. For 8-bit dividends, four iterations
are sufficient to reproduce the exact integer quotient over the entire range. For 16-bit dividends,
eight iterations suffice.

One subtle issue remains: every time we shift right, we throw away fractional information. Left unchecked,
these tiny losses accumulate and the final result drifts low. The solution is to preload the accumulator
with a carefully chosen constant before iteration begins. The preload biases the iteration upward to compensate
for the downward truncation introduced by repeated right shifts. For an 8-bit dividend, the preload constant
is floor(256 / 3). For a 16-bit dividend, it is floor(65536 / 3).

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

The `%11111110` mask is used because it clears the least significant bit and resets the carry flag,
ensuring that subsequent `rra` instructions behave like chained logical shifts, which would otherwise
require slower `srl a` operations.

## Divisors of the form 2^N + 1

Another class of elegant routines emerges for divisors that are one greater than a power of two:

```
 5: y = (x - y) >> 2
 9: y = (x - y) >> 3
17: y = (x - y) >> 4
...
```

At first glance, this looks almost identical to the addition-based recurrence of the form y = (x + y) >> N.
However, there is an important difference: unlike the addition form, the truncation error now oscillates in sign,
so the rounding bias tends to cancel out naturally and no preload constant is required.

For example, division by 5 can be implemented directly as:

y = (x - y) >> 2

On the Z80 we can exploit the identity:

-y = ~y + 1

because complement (`cpl`) is cheaper than negation (`neg`).
The additional increment can be absorbed by pre-incrementing the input value before iteration begins.
The resulting division-by-5 (A = B / 5) routine becomes:

```
ld    a,b
inc   b
srl   a
srl   a
cpl
add   a,b
rra
srl   a
cpl
add   a,b
rra
srl   a
cpl
add   a,b
rra
srl   a
```

## Closing Thoughts

These routines are not replacements for general-purpose division. They are specialized iterative solvers
for particular divisors, especially numbers near powers of two. But they represent an interesting and
philosophically satisfying alternative to restoring division: instead of extracting quotient bits one at a time,
they converge toward the quotient using only shifts, additions, complements, and feedback.
