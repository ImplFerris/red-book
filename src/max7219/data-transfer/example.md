# Example: Writing to a Digit

Just like we turned the display on by writing to the Shutdown register, we can send data to the digit registers to control what appears. The meaning of the data byte depends on whether we are using a 7-segment display or an 8x8 LED matrix.

## 7-segment display

In a 7-segment display, the digit registers `0x01` to `0x08` each control one digit. The value you send is a segment code that maps to segments A to G and the decimal point. For example, to show the number `5` on Digit 0 we send:

> Note: If the MAX7219's Code B decode mode is enabled (in the Decode Mode register), you can send digit values (0-9) directly to the digit registers. Otherwise, you need to send the segment bit pattern manually.

```rust
[0x01, 0x05]    // 16-bit value 0x0105 represented as a byte array (this is how we actually do it in the code)
```

This means: write the code `0x05` (pattern for 5) into the Digit 0 register.

## LED Matrix

In an LED matrix, the digit registers `0x01` to `0x08` represent rows 0 to 7 of the display. The value you send is an 8-bit pattern, where each bit controls one LED in that row. We will use binary to make it clear which LEDs are on.

For example, to turn on the far left and far right LEDs of Row 0 we send:

```rust
[0x01, 0x81]
```

`0x81` in binary is:

```
0x81 = 10000001
```

Each 1 means the LED at that bit position is on, and each 0 means it is off. In this case, bit 7 and bit 0 are set, so the leftmost and rightmost LEDs in Row 0 are lit.

A typical bit position to column mapping is:

```
bit 7 -> column 0 (leftmost)
bit 6 -> column 1
bit 5 -> column 2
bit 4 -> column 3
bit 3 -> column 4
bit 2 -> column 5
bit 1 -> column 6
bit 0 -> column 7 (rightmost)
```

So `10000001` lights columns 0 and 7.

```
Row 0:  ■  .  .  .  .  .  .  ■
Row 1:  .  .  .  .  .  .  .  .
Row 2:  .  .  .  .  .  .  .  .
Row 3:  .  .  .  .  .  .  .  .
Row 4:  .  .  .  .  .  .  .  .
Row 5:  .  .  .  .  .  .  .  .
Row 6:  .  .  .  .  .  .  .  .
Row 7:  .  .  .  .  .  .  .  .
```
