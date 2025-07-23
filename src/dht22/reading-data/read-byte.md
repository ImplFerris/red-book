# Read Byte

Now let's implement the read_byte function. This function reads 8 bits one after another and combines them into a single u8 byte.

```rust
 fn read_byte(&mut self) -> Result<u8, DhtError<E>> {
    let mut byte: u8 = 0;

    for i in 0..8 {
        let bit_mask = 1 << (7 - i);
        if self.read_bit()? {
            byte |= bit_mask;
        }
    }

    Ok(byte)
}
```
We start by initializing the byte to 0. Then, in a loop that runs 8 times (once for each bit), we calculate a bit mask for the current position using 1 << (7 - i) so that the first bit read goes into the highest bit position.

For each iteration, we call read_bit() to get the next bit from the sensor. If the bit is 1, we use the bitwise OR operation to set the corresponding bit in the byte. If it is 0, we leave the bit as-is (it's already 0). After all 8 bits are read and assembled, the final byte is returned. This approach matches how the DHT22 sends data: one bit at a time, from the most significant bit to the least significant bit.


## Testing read_byte function

To simulate the sensor's behavior, we use a helper function called encode_byte, which takes a byte like 0b10111010 and expands it into a sequence of PinTx transactions for each of the 8 bits. Each bit involves four pin states: first low (bit start), then high (bit signal), then the actual sample level (high for 1, low for 0), and finally low again (end of bit).

```rust
// Helper to encode one byte into 8 bits (MSB first)
fn encode_byte(byte: u8) -> Vec<PinTx> {
    (0..8)
        .flat_map(|i| {
            // Extract bit (MSB first: bit 7 to bit 0)
            let bit = (byte >> (7 - i)) & 1;
            vec![
                PinTx::get(PinState::Low),  // wait_for_low
                PinTx::get(PinState::High), // wait_for_high
                PinTx::get(if bit == 1 {
                    // sample
                    PinState::High
                } else {
                    PinState::Low
                }),
                PinTx::get(PinState::Low), // end of bit
            ]
        })
        .collect()
}
```

We then set up a PinMock with these transactions and prepare the delay mock with eight 35us delays; one for each bit sample timing. When read_byte() is called, it internally calls read_bit() eight times, and each read_bit() checks the pin after a 35us delay to determine the bit value.


```rust
#[test]
fn test_read_byte() {
    let pin_states = encode_byte(0b10111010);

    let mut pin = PinMock::new(&pin_states);
    let delay_expects = vec![DelayTx::delay_us(35); 8];
    let mut delay = CheckedDelay::new(&delay_expects);

    let mut dht = Dht22::new(pin.clone(), &mut delay);
    let byte = dht.read_byte().unwrap();
    assert_eq!(byte, 0b10111010);

    pin.done();
    delay.done();
}
```

We check that the byte returned by read_byte() is exactly 0b10111010, the same one we encoded into the mock. This confirms that our function correctly reads each bit in order and assembles them into a byte. At the end, we call done() on both the pin and delay mocks to make sure all the expected steps actually happened.
