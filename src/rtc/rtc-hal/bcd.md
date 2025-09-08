# Binary Coded Decimal(BCD) utilities

We won't declare any traits here - instead we'll provide utility functions for BCD operations. Most RTC chips use BCD format to store time and date values.

## Binary Code Decimal(BCD) 

If I ask you the value of binary "00010010", you will do the usual math (or use a programmer calculator) and say it is 18 in decimal (or 0x12 in hex). But in BCD format, this same binary represents decimal 12, not 18!

Don't let hex looking same as BCD fool you, there is a difference. Let me show you.

### How BCD Works

BCD uses 4 bits for each decimal digit. So if you want to show decimal 12, you split it into two digits (1 and 2) and convert each one separately.

```sh
Decimal:    1    2
BCD:     0001 0010
Bits:    8421 8421
```

To read this:

First 4 bits (0001) = 1
Second 4 bits (0010) = 2

Here, the first part represents the tens of a number. So we have to multiply it by 10. The second part represents the ones of a number

Put together: (1 × 10) + (2 * 1) = 12

Let's do another example to make it clear. Here is how decimal 36 represented as

```sh
Decimal:    3    6
BCD:     0011 0110  
Bits:    8421 8421
```

To read this:

First 4 bits (0011) = 3
Second 4 bits (0110) = 6

Put together: (3 × 10) + 6 = 36

### BCD vs Hex

Here's the tricky part. The binary 0011 0110 looks like hex 0x36 (which is decimal 54). But in BCD, it means decimal 36.

The big difference: BCD only allows 0-9 in each 4-bit group.

Hex can go from 0-15 (shown as 0-9, A-F), but BCD stops at 9. This makes sense because each 4-bit group represents one decimal digit, and single digits only go 0 to 9.

**What Happens When You Break the Rules**

```sh
Binary:  0011 1010
Hex:     0x3A (decimal 58)
BCD:     Not valid!
```

This doesn't work in BCD because the second group (1010) is 10, which isn't a single digit. Hex is fine with this (0x3A), but BCD says no.

BCD simplifies displaying decimal numbers (for example, on a 7-segment display driven by a MAX7219). Each 4-bit group directly represents a single decimal digit.
