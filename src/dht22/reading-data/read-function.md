# Final Boss

We have now implemented all the building blocks of our DHT22 driver. The last piece is the read function, which brings everything together and is the main function users will call to get a sensor reading. 

First, it sends the start signal to the DHT22. Then, it reads four bytes of data, which include the humidity and temperature values. After that, it reads one more byte, which is the checksum. The checksum is simply the sum of the previous four bytes, and it's used to detect transmission errors. If the calculated checksum doesn't match the one sent by the sensor, we return a ChecksumMismatch error. Otherwise, we pass the data to our parse_data function to convert it into a Reading.

```rust
 pub fn read(&mut self) -> Result<Reading, DhtError<E>> {
    self.start()?;

    let mut data = [0; 4];

    for b in data.iter_mut() {
        *b = self.read_byte()?;
    }

    let checksum = self.read_byte()?;
    if data.iter().fold(0u8, |sum, v| sum.wrapping_add(*v)) != checksum {
        Err(DhtError::ChecksumMismatch)
    } else {
        Ok(self.parse_data(data))
    }
}
```

## Testing the read function

Now we test the complete behavior of the read function from start to finish. The sensor is expected to send four data bytes followed by a checksum byte. In this case, the values represent 40.0% humidity and 24.6°C temperature. We also include the required delays: one for the initial start request, and then one 35 microsecond delay for each of the 40 data bits.

```rust
#[test]
fn test_read_valid() {
    // Data to simulate: [0x01, 0x90, 0x00, 0xF6], checksum = 0x87

    // Start sequence
    let mut pin_states = start_sequence();

    let data_bytes = [0x01, 0x90, 0x00, 0xF6];
    let checksum = 0x87;

    for byte in data_bytes.iter().chain(std::iter::once(&checksum)) {
        pin_states.extend(encode_byte(*byte));
    }

    let mut pin = PinMock::new(&pin_states);

    // Delays: start = 1ms + 40us
    let mut delay_transactions = vec![DelayTx::delay_ms(1), DelayTx::delay_us(40)];
    // Delay for data bit transfer: 40 bits * 35us delay
    delay_transactions.extend(std::iter::repeat_n(DelayTx::delay_us(35), 40));

    let mut delay = CheckedDelay::new(&delay_transactions);

    let mut dht = Dht22::new(pin.clone(), &mut delay);
    let reading = dht.read().unwrap();

    assert_eq!(
        reading,
        Reading {
            relative_humidity: 40.0,
            temperature: 24.6,
        }
    );

    pin.done();
    delay.done();
}
```

The test_read_invalid function is similar, but instead of using the correct checksum (0x87), we intentionally provide an incorrect one (0x81). This test checks whether the driver correctly detects a checksum mismatch and returns an appropriate error. Both tests use pin and delay mocks to simulate real hardware behavior without requiring an actual sensor, allowing us to fully test the driver's logic in isolation.


```rust
#[test]
fn test_read_invalid() {
    // Data to simulate: [0x01, 0x90, 0x00, 0xF6], checksum = 0x87

    // Start sequence
    let mut pin_states = start_sequence();

    let data_bytes = [0x01, 0x90, 0x00, 0xF6];
    let checksum = 0x81; // Wrong checksum value

    for byte in data_bytes.iter().chain(std::iter::once(&checksum)) {
        pin_states.extend(encode_byte(*byte));
    }

    let mut pin = PinMock::new(&pin_states);

    // Delays: start = 1ms + 40us
    let mut delay_transactions = vec![DelayTx::delay_ms(1), DelayTx::delay_us(40)];
    // Delay for data bit transfer: 40 bits * 35us delay
    delay_transactions.extend(std::iter::repeat_n(DelayTx::delay_us(35), 40));

    let mut delay = CheckedDelay::new(&delay_transactions);

    let mut dht = Dht22::new(pin.clone(), &mut delay);
    assert_eq!(dht.read().unwrap_err(), DhtError::ChecksumMismatch);

    pin.done();
    delay.done();
}
```

That's all, folks! We finally did it. Our very first embedded Rust driver is complete.
