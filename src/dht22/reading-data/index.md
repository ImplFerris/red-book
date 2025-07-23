# Reading Data

The user of our driver will basically call dht.read() to get the sensor data. This function returns a Reading struct that contains both temperature and humidity values. 

When you call read(), it first sends a start signal to the sensor. Then it waits for the DHT22 to respond. After that, it reads 5 bytes from the sensor. This reading happens through the read_byte() function, which in turn reads 8 individual bits using read_bit().

Once all the bytes are received, the driver checks whether the checksum (5th byte) matches the sum of the first 4 bytes. If the data is valid, it then parses the values and returns the final Reading.

Basically, these are the functions that read() will eventually call internally:
```rust
read() --> start()
       --> read_bytes() --> uses read_bit()
       --> checksum check
       --> parse_data() --> Reading
```

We will not define the read function yet. We will define the dependent functions first. That way, we can run tests on them directly. Otherwise, we would have to put todo!() or deal with compilation errors. Instead, we finish each of these individual parts and finally integrate them together in the read() function.
 
## Start Sequence

Before the sensor sends any data, the microcontroller must initiate the communication by sending a start signal. This is done by pulling the data line low for at least 1 ms, then pulling it high again and waiting briefly(20-40 us). After that, the DHT22 will respond with a specific low-high pattern, confirming it's ready to send data.

Here's how we implement that logic in the start() function:

```rust
fn start(&mut self) -> Result<(), DhtError<E>> {
    // MCU sends start request
    self.pin.set_low()?;
    self.delay.delay_ms(1);
    self.pin.set_high()?;
    self.delay.delay_us(40);

    // Waiting for DHT22 Response
    self.wait_for_low()?
    self.wait_for_high()?; 
    Ok(())
}
```

## Testing the Start sequence

Inside the mod tests, we will first define a helper function. This returns the expected pin transactions for the start sequence. Since we will use this same pattern in multiple tests, it makes sense to extract it once here.

```rust
 fn start_sequence() -> Vec<PinTx> {
    vec![
        // MCU initiates communication by pulling the data line low, then releasing it (pulling it high)
        PinTx::set(PinState::Low),
        PinTx::set(PinState::High),
        // Sensor responds
        PinTx::get(PinState::Low),
        PinTx::get(PinState::High),
    ]
}
```

Now we test the start function. If you look at the start implementation and the expected pin transaction sequence above, they will match step-by-step.

We are sending a 1 millisecond delay after pulling the pin low, so the mock must expect a delay_ms(1) transaction. Then we send a 40 microsecond delay after setting the pin high, so the mock must also expect a delay_us(40) transaction. 

When we call wait_for_low() and wait_for_high(), we immediately return the expected pin state in the mock setup, so no delay is triggered for those steps.

```rust
 #[test]
    fn test_start_sequence() {
        let mut expect = vec![];
        expect.extend_from_slice(&start_sequence());

        let mut pin = PinMock::new(&expect);

        let delay_transactions = vec![DelayTx::delay_ms(1), DelayTx::delay_us(40)];
        let mut delay = CheckedDelay::new(&delay_transactions);

        let mut dht = Dht22::new(pin.clone(), &mut delay);
        dht.start().unwrap();

        pin.done();
        delay.done();
    }
```
