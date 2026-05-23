# A Deferred Approach to Signed Shift-and-Add Multiplication

*By Milos "baze" Bazelides*

The standard shift-and-add multiplication algorithm assumes every bit contributes to the number’s magnitude.
That works perfectly for unsigned integers, but it breaks down for signed values in two’s complement form.
If we feed a negative number directly into the algorithm, the sign bit is treated as an ordinary magnitude
bit, producing a wrong result.

A common workaround is to first convert the operation into an unsigned multiplication problem:

1. Check the signs of both operands (typically using `xor`) to determine whether the result should be negative.
2. Convert any negative operands to their absolute values.
3. Perform a standard unsigned shift-and-add multiplication.
4. If the recorded sign indicates the result should be negative, negate the final product.

This approach works, but it adds extra bookkeeping and conditional logic around what is otherwise a simple
algorithm. To see what such an unsigned algorithm looks like, consider the multiplication of two 8-bit
values held in registers C and E:
```
; Input: C = first operand, E = second operand
; Output: HL = product

      ld    h,c
      ld    l,0
      ld    d,l
      ld    b,8
Mul   add   hl,hl
      jr    nc,Skip
      add   hl,de
Skip  djnz  Mul
```

Could we avoid bookkeeping entirely and defer the correction until after the multiplication completes? The key insight
is how the unsigned multiplier interprets the most significant bit: instead of contributing −128, it contributes +128.
Whenever a sign bit is set, the multiplier overestimates its value by 256. The error term introduced by a negative A
is therefore 256 * B, and the same logic applies to A when B is negative.

In practice, this reduces to a very simple adjustment based on the sign bits: if one operand is negative, subtract the
other operand from the high byte of the product:
```
; Input: C = first operand, E = second operand
; Output: HL = product

      ld    h,c
      ld    l,0
      ld    d,l
      ld    b,8
Mul   add   hl,hl
      jr    nc,Skip
      add   hl,de
Skip  djnz  Mul

      ld    a,h
      bit   7,c
      jr    z,Fix1
      sub   e
Fix1  bit   7,e
      jr    z,Fix2
      sub   c
Fix2  ld    h,a
```
