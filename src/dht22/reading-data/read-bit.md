# Read Bit

Let's implement the read_bit function. If you remember the DHT protocol we explained earlier, the DHT sensor always pulls the line low to indicate the start of each bit. Then it switches the line to high, and the duration of that high signal determines whether the bit is a 0 or a 1.

In the DHT protocol, if the high signal lasts around 26-28 microseconds, the bit is considered to be 0. If the high duration is more than 70 microseconds, it is considered to be 1.

Some implementations try to measure exactly how long the signal stays high, and based on that, decide if the bit is 0 or 1. I also thought about doing this. But there is a much simpler approach, used by the dht-sensor crate. 

Instead of measuring the time precisely, we just wait for 35 microseconds and then check the pin. If the pin is still high after 35us, that means the DHT22 kept the signal high for more than 28 microseconds, which indicates a 1. On the other hand, if the pin has already gone low before we check, then the high duration was short, which means the bit is a 0 (because for a bit to be 1, the high signal must last at least around 70 microseconds).

Here is the implementation:

```rust
fn read_bit(&mut self) -> Result<bool, DhtError<E>> {
    // Wait for DHT pulls line low
    self.wait_for_low()?; // ~50us

    // Step 2: DHT pulls line high
    self.wait_for_high()?;

    // Step 3: Delay ~35us, then sample pin
    self.delay.delay_us(35);

    // If it is still High, then the bit value is 1
    let bit_is_one = self.pin.is_high()?;
    self.wait_for_low()?;

    Ok(bit_is_one)
}
```

## Testing the read_bit Function

We are now testing the read_bit function, which determines whether a bit from the DHT22 is a 1 or a 0 based on how long the signal stays high.

There are no new concepts here. The way we construct the expected pin and delay transactions is exactly like we did before. We're simulating the behavior of the DHT22 using mock pins and mock delays. That way, we can test the logic without actual hardware.

We have two tests:

### Test for Bit Value 1

In this case, we simulate the DHT pulling the line low, then high, and keeping it high long enough that when we sample the pin after 35 microseconds, it is still high. That tells us the bit value is 1.

```rust
 #[test]
fn test_read_bit_one() {
    let mut pin = PinMock::new(&[
        // wait_for_low
        PinTx::get(PinState::Low), // Mimicks DHT pulling low to signal start of data bit
        // wait_for_high
        PinTx::get(PinState::High), // Then pulls high - duration determines bit value
        // delay_us(35) -> handled in delay
        // Sample pin after delay
        PinTx::get(PinState::High), // is it still High? (High -> 1)
        // Final wait_for_low
        PinTx::get(PinState::Low), // End of bit
    ]);

    let delay_transactions = vec![
        // wait_for_low
        DelayTx::delay_us(35),
    ];
    let mut delay = CheckedDelay::new(&delay_transactions);

    let mut dht = Dht22::new(pin.clone(), &mut delay);

    let bit = dht.read_bit().unwrap();
    assert!(bit);

    pin.done();
    delay.done();
}
```

### Test for Bit Value 0

This one is a bit more elaborate. I admit it's a bit overdone for what we need, but it works. The idea is to simulate the signal going high and then falling back low before the 35us delay is over. That indicates a short high pulse, so the bit is 0.

```rust
#[test]
fn test_read_bit_zero() {
    let mut pin = PinMock::new(&[
        // wait_for_low
        PinTx::get(PinState::High), // To trigger Delay of 1 us, we keep it High first
        PinTx::get(PinState::Low),
        // wait_for_high
        PinTx::get(PinState::Low), // To trigger Delay of 1 us, we keep it Low first
        PinTx::get(PinState::High), // now high
        // sample bit after delay (35us)
        PinTx::get(PinState::Low), // We will set it Low to indicate bit value is "0"
        // final wait_for_low
        PinTx::get(PinState::High), // To trigger Delay of 1 us, we keep it High first
        PinTx::get(PinState::Low),  // now low
    ]);

    let delay_transactions = vec![
        DelayTx::delay_us(1),  // after 1st pin high during wait_for_low
        DelayTx::delay_us(1),  // after 1st pin low during wait_for_high
        DelayTx::delay_us(35), // sampling delay
        DelayTx::delay_us(1),  // after 1st high in final wait_for_low
    ];
    let mut delay = CheckedDelay::new(&delay_transactions);

    let mut dht = Dht22::new(pin.clone(), &mut delay);

    let bit = dht.read_bit().unwrap();
    assert!(!bit);

    pin.done();
    delay.done();
}
```

### Test timeout
This test checks whether the driver correctly handles a timeout situation. We simulate a case where the DHT22 never pulls the signal low by keeping the pin state high for a long time. This should trigger a timeout in the read_bit function, which waits for a state change. 

We provide 100 high states and 100 microsecond delays to simulate this. When read_bit doesn't detect the expected transition in time, it returns a Timeout error. The test then asserts that this error is indeed returned. 

```rust
#[test]
fn test_read_timeout() {
    let pin_expects: Vec<PinTx> = (0..100).map(|_| PinTx::get(PinState::High)).collect();
    let mut pin = PinMock::new(&pin_expects);

    let delay_expects: Vec<DelayTx> = (0..100).map(|_| DelayTx::delay_us(1)).collect();

    let mut delay = CheckedDelay::new(&delay_expects);

    let mut dht = Dht22::new(pin.clone(), &mut delay);

    assert_eq!(dht.read_bit().unwrap_err(), DhtError::Timeout);

    pin.done();
    delay.done();
}
```
