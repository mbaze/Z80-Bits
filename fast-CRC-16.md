# A Fast Z80 Implementation of CRC-16

*By Milos "baze" Bazelides (2026-05-19)*

The routine shown below implements a fast, table-driven, left-shifting CRC-16 on the Z80 using
two 256-byte lookup tables. For each incoming data byte it performs the following operations:
```
uint8_t index = input_byte ^ crc_hi;
crc_lo = crc_table_lo[index];
crc_hi = crc_lo ^ crc_table_hi[index];
```

A traditional CRC-16 implementation processes data one bit at a time, performing up to eight
conditional `xor` operations per byte. The table-driven version accelerates this by precomputing
the effect of all possible 8-bit inputs. The 16-bit CRC state is maintained as two 8-bit values
(`crc_hi` and `crc_lo`), allowing the Z80 to update the CRC using only `xor`s and table lookups
instead of performing the calculation bit by bit. In the code below, register A corresponds to
`crc_hi` and register A' corresponds to `crc_lo`:
```
; Input:
; HL = input block address
; BC = block length
; DE = expected CRC

; Output:
; A:A' = CRC
; ZF set on match

Crc16      exx
           ld    d,CRC_TABS / 256
           ld    h,CRC_TABS / 256 + 1
           exx
           xor   a
           ex    af,af'
           xor   a

CrcLoop    xor   (hl)
           exx
           ld    e,a
           ld    l,a
           ld    a,(de)
           ex    af,af'
           xor   (hl)
           exx
           cpi
           jp    pe,CrcLoop

           cp    d
           ret   nz
           ex    af,af'
           cp    e
           ret
```

For convenience, a lookup table generator is also provided. The tables must be aligned to a
256-byte boundary:
```
CRC_POLY   equ   #A2EB
CRC_TABS   equ   #FE00

Crc16Gen   ld    c,0
CrcLoop1   ld    h,0
           ld    l,c
           ld    b,16
CrcLoop2   add   hl,hl
           jr    nc,CrcNext
           ld    a,l
           xor   CRC_POLY % 256
           ld    l,a
           ld    a,h
           xor   CRC_POLY / 256
           ld    h,a
CrcNext    djnz  CrcLoop2
           ld    b,CRC_TABS / 256
           ld    a,l
           ld    (bc),a
           inc   b
           ld    a,h
           ld    (bc),a
           inc   c
           jr    nz,CrcLoop1
           ret
```

The polynomial used is CRC-16F/4.2, taken from Philip Koopman’s excellent [CRC Polynomial Zoo]
(https://users.ece.cmu.edu/~koopman/crc/). It is particularly well suited to packet sizes around
4 KiB (32768 bits), where it achieves a Hamming distance of HD = 4 up to 32751 bits of message
length. In practice, this guarantees detection of all 1-bit, 2-bit, and 3-bit errors, while also
providing strong burst-error detection characteristics.

To use the standard CRC-CCITT polynomial instead, set `CRC_POLY = #1021`.
