# Data Bits

Once the DHT22 completes its initial response, it sends a total of 40 bits of data to the microcontroller. These 40 bits are structured as five consecutive bytes, transmitted one bit at a time over the single-wire bus. The data is sent most significant bit (MSB) first.

## Relative Humidity Value
The first two bytes represent the humidity value. The first byte is the high byte (most significant 8 bits) and the second is the low byte. Together, they form a 16-bit unsigned integer. To convert this to a percentage, the value must be divided by 10. For example, if the bytes are `0x01` and `0x90`, the raw 16-bit value is `0x0190`, which equals 400 in decimal. Dividing by 10 gives a humidity reading of 40.0%.

## Temperature Value
The next two bytes represent the temperature, again as a 16-bit value with the high byte first. This value is also divided by 10 to get the temperature in Celsius. For instance, if the temperature bytes are `0x00` and `0xFA`, the combined value is `0x00FA` or 250 in decimal, giving a temperature of 25.0°C. If the most significant bit of the temperature high byte is set (i.e., bit 15 is 1), it indicates a negative temperature. In that case, the temperature value should be interpreted as a signed 16-bit number, and the final result divided by 10 will be negative.

## Checksum
The fifth and final byte is a checksum. This is used to verify that the data was received correctly. The checksum is calculated by summing the first four bytes and taking only the least significant 8 bits of the result (i.e., modulo 256). If the checksum sent by the sensor matches the calculated value, the data is considered valid.

Here is an example of a full 40-bit transmission represented in ASCII bit layout:

```text
+--------+--------+--------+--------+--------+
|00000001|10010000|00000000|11111010|10001011|
+--------+--------+--------+--------+--------+
HumidH   HumidL   TempH    TempL     CRC

(0x01)   (0x90)   (0x00)   (0xFA)   (0x8B)
```

In this example, the humidity is 40.0%, the temperature is 25.0°C, and the checksum byte 0x8B matches the calculated sum:
0x01 + 0x90 + 0x00 + 0xFA = 0x18B. 

Since the checksum is only one byte (8 bits), we only care about the lower 8 bits of this sum. To extract them, we apply a bitwise AND with 0xFF, which keeps just the least significant byte: 0x18B & 0xFF = 0x8B. This result matches the checksum sent by the sensor; on the microcontroller side(in our driver), we can perform this check to ensure the data was received correctly.

In Rust, instead of summing the values as a larger type like u16 and then applying a bitwise AND with 0xFF, we can just use `wrapping_add` function, which automatically keeps the result within 8 bits by discarding any overflow. Since our incoming data is already u8, using wrapping_add is more efficient and better suited for our case.

For example:
```rust
fn main() {
    let data: [u8; 4] = [0x01, 0x90, 0x00, 0xFA];
    let expected_checksum: u8 = 0x8B;

    let calculated_checksum = data.iter().fold(0u8, |sum, v| sum.wrapping_add(*v));
    println!("calculated checksum: 0x{:02X}", calculated_checksum);
    println!("is valid: {}", expected_checksum == calculated_checksum);
}
```

## Handling Negative Temperatures

The DHT22 encodes negative temperatures using the most significant bit (MSB) of the 16-bit temperature value as a sign bit. If this bit is set (i.e., bit 15 of the 16-bit value is `1`), the temperature is negative. To extract the correct value, you must first detect the sign, then mask off the sign bit and apply the negative sign after conversion.

For example, suppose the DHT22 returns the following two bytes for temperature:

```text
Temp High Byte: 10000000 (0x80)
Temp Low Byte:  00001010 (0x0A)
```

Combined, these form the 16-bit value 0x800A. The MSB is 1, indicating a negative temperature.

To compute the temperature:

Mask off the sign bit: 0x800A & 0x7FFF = 0x000A 

```text
Mask off sign bit calculation represented with binary for better clarity

   10000000 00001010   (original, 0x800A)
&  01111111 11111111   (mask,     0x7FFF)
----------------------
   00000000 00001010   (result,   0x000A)
```

Convert to decimal: 0x000A = 10

Divide by 10: 10 / 10 = 1.0

Apply the sign: -1.0 °C

So, this bit sequence indicates a temperature of –1.0 °C.


