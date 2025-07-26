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
We start by initializing the byte to 0. Then, in a loop that runs 8 times (once for each bit), we calculate a [bit mask](./read-byte.html#optional-how-read_byte-uses-bitmasks-to-build-a-byte) for the current position using 1 << (7 - i) so that the first bit read goes into the highest bit position.

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

## [Optional] How read_byte Uses Bitmasks to Build a Byte

If you're not sure how the bitmask and the loop work together to build the byte, this appendix will explain it step by step. If you already know this, feel free to skip this.

The read_byte function reads 8 bits from the DHT22 sensor and builds a u8 by placing each bit in its correct position. It starts from the most significant bit (bit 7) and moves down to the least significant bit (bit 0).

### Step by Step Bitmask Calculation 

We will assume the bits received from the sensor are: 1, 0, 1, 1, 1, 0, 1, 0 (which is 0b10111010).

We start with a byte set to all zeros:
```rust
let mut byte: u8 = 0; //0b00000000 in Rust binary notation
```

Now, for each bit we read, we calculate a bit mask by shifting 1 to the left by (7 - i) positions. This places the "1" in the correct position for that bit inside the final byte.

Let's go step by step:

#### Iteration 0: Received bit = 1

We shift 1 by 7 places to the left to reach the most significant bit. Since, we received the value "1" from the sensor for this position, so we enter the `if` block and use the OR (|) operator to set that corresponding bit in the byte.

The current value of the "byte" is 0b00000000. When we apply the OR operation with the bit mask, the result becomes 0b10000000.

```rust
i = 0: bit = 1
bit_mask = 1 << (7 - 0) = 0b10000000
byte |= 0b10000000 => 0b10000000
```

Here's how the OR operation sets the bit in the correct position:

| Bit Position | 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
| ------------ | - | - | - | - | - | - | - | - |
| `byte`       | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| `bit_mask`   | 1 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| `result`     | 1 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |


#### Iteration 1: Received bit = 0

We received the bit value "0" from the sensor. Since it's already zero, there's no need to update the byte. The `if` block is skipped, and the byte remains unchanged.

```text
i = 1: bit = 0
(skip update since bit is already 0)
```

#### Iteration 2: Received bit = 1

We shift 1 by 5 places to create a mask for bit 5. Since we received a 1, we update the byte using the OR operation. The current byte is 0b10000000, and after the `OR` operation, it becomes 0b10100000.

```rust
i = 2: bit = 1
bit_mask = 1 << (7 - 2) = 0b00100000
byte |= 0b00100000 => 0b10100000
```

Bitwise OR Operation Breakdown:

```text
  10000000
| 00100000
------------
  10100000
```


#### Iteration 3: Received bit = 1

We shift 1 by 4 places. The bit is 1, so we set bit 4.

```rust
i = 3: bit = 1
bit_mask = 1 << (7 - 3) = 0b00010000
byte |= 0b00010000 => 0b10110000
```

This is the same as the previous bitwise OR operations we did. To think of it simply, we are turning bit 4 to 1 without affecting the other bits.


#### Iteration 4: Received bit = 1

Same as before, this sets the bit at position 3 to the value 1. Other bits remain unchanged.

```rust
i = 4: bit = 1
bit_mask = 1 << (7 - 4) = 0b00001000
byte |= 0b00001000 => 0b10111000
```

#### Iteration 5: Received bit = 0

This bit is 0, so no update is made.

```rust
i = 5: bit = 0
(skip update since bit is 0)
```

#### Iteration 6: Received bit = 1

Same as before, this sets the bit at position 1 to the value 1. Other bits remain unchanged.

```rust
i = 6: bit = 1
bit_mask = 1 << (7 - 6) = 0b00000010
byte |= 0b00000010 => 0b10111010
```

#### Iteration 7: Received bit = 0

Since we received bit value "0", so the we wont update the byte.

```rust
i = 7: bit = 0
(skip update since bit is 0)
```

The final result after processing all 8 bits is `0b10111010`. This binary value is equal to 0xBA in hexadecimal and 186 in decimal.
