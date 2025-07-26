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
We start by initializing the byte to 0. Then, in a loop that runs 8 times (once for each bit), we calculate a [bit mask](./#how-read_byte-builds-a-byte-from-bits) for the current position using 1 << (7 - i) so that the first bit read goes into the highest bit position.

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

---

## How read_byte Builds a Byte from Bits

In case you're not sure how the bitmask and the whole loop construct the byte, i am adding this appendix explanation for you.

The read_byte function reads 8 bits from the DHT22 sensor and builds a u8 by placing each bit in its correct position. It starts from the most significant bit (bit 7) and moves down to the least significant bit (bit 0).

### Step by Step Bitmask Calculation 

We will assume the bits received from the sensor are: 1, 0, 1, 1, 1, 0, 1, 0 (which is 0b10111010).

We start with a byte set to all zeros:
```rust
let mut byte: u8 = 0; //0b00000000
```

Now, for each bit we read, we calculate a bit mask by shifting 1 to the left by (7 - i) positions. This places the 1 in the correct position for that bit inside the final byte.

Let's go step by step:

**Iteration 0: Received bit = 1**

We shift 1 by 7 places to the left to reach the most significant bit. Since bit is 1, we set that bit using |= operator.

```rust
i = 0: bit = 1
bit_mask = 1 << (7 - 0) = 0b10000000
byte |= 0b10000000 => 0b10000000
```

**Iteration 1: Received bit = 0**

This bit is 0, so it won't go into the if condition and we skip updating the byte.
```rust
i = 1: bit = 0
(skip update since bit is already 0)
```

**Iteration 2: Received bit = 1**

We shift 1 by 5 places. Since the bit is 1, we set bit 5 using the mask.

```rust
i = 2: bit = 1
bit_mask = 1 << (7 - 2) = 0b00100000
byte |= 0b00100000 => 0b10100000
```

**Iteration 3: Received bit = 1**

We shift 1 by 4 places. The bit is 1, so we set bit 4.

```rust
i = 3: bit = 1
bit_mask = 1 << (7 - 3) = 0b00010000
byte |= 0b00010000 => 0b10110000
```

**Iteration 4: Received bit = 1**

We shift 1 by 3 places. Bit is 1, so we update bit 3.

```rust
i = 4: bit = 1
bit_mask = 1 << (7 - 4) = 0b00001000
byte |= 0b00001000 => 0b10111000
```

**Iteration 5: Received bit = 0**

This bit is 0, so no update is made.

```rust
i = 5: bit = 0
(skip update since bit is 0)
```

**Iteration 6: Received bit = 1**

We shift 1 by 1 place to target bit 1. Bit is 1, so we update bit 1.

```rust
i = 6: bit = 1
bit_mask = 1 << (7 - 6) = 0b00000010
byte |= 0b00000010 => 0b10111010

```

**Iteration 7: Received bit = 0**

Bit is 0, so the loop skips the update.

```rust
i = 7: bit = 0
(skip update since bit is 0)
```

The final result after processing all 8 bits is 0b10111010. This binary value is equal to 0xBA in hexadecimal and 186 in decimal. So, the final byte returned by the function is 186. This example shows how each bit is positioned using bitmasks and combined using |= to gradually build up the final value.
