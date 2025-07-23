# Writing Tests for Embedded Rust Using Embedded HAL Mock

Before moving further into the chapter, let me introduce a nice crate: [`embedded-hal-mock`](https://docs.rs/embedded-hal-mock/latest/embedded_hal_mock/). This crate lets us mock the Embedded HAL traits. The implementations don't touch any real hardware. Instead, they use mock or no-op versions of the traits.

The idea is simple: we can write and run tests for our drivers, even in CI, without needing real hardware. We'll use this crate to test our DHT22 driver.

## Dev Dependency

To use embedded-hal-mock for testing, we'll add it as a dev-dependency in Cargo.toml. A dev-dependency is a crate that's only needed while testing or building examples. It won't be included when someone uses your library as a dependency. 

Add this lines to your Cargo.toml:

```toml
[dev-dependencies]
embedded-hal-mock = { version = "0.11.1", default-features = false, features = [
    # eh1: Provide module eh1 that mocks embedded-hal version 1.x (enabled by default)
    "eh1",
] }
```

We are using embedded-hal 1.x, so we'll enable the "eh1" feature.

With embedded-hal-mock, we can mock things like pin states, I2C, SPI, PWM, delay, and serial. For our driver, we'll be using the pin and delay mocks.


## Writing Our First Test

Let's write our first test using embedded-hal-mock. This test checks if our `wait_for_high` and `wait_for_low` helpers work as expected. We simulate a pin that stays low for two cycles before going high, then goes low again. The delay_us calls simulate tiny pauses between checks.

Final code for the first test is shown below. You should place this inside your dht22.rs module. Don't worry if it looks overwhelming right now. In the upcoming section, we will go through it step by step and explain how everything works.

```rust

#[cfg(test)]
mod tests {
    use super::*;
    use embedded_hal_mock::eh1::delay::CheckedDelay;
    use embedded_hal_mock::eh1::delay::Transaction as DelayTx;
    use embedded_hal_mock::eh1::digital::{
        Mock as PinMock, State as PinState, Transaction as PinTx,
    };
    
    #[test]
    fn test_wait_for_state() {
        let mut expect = vec![];

        expect.extend_from_slice(&[
            // pin setting high
            PinTx::set(PinState::High),
            // wait_for_high
            PinTx::get(PinState::Low), // Triggers Delay 1us
            PinTx::get(PinState::Low), // Triggers Delay 1us
            PinTx::get(PinState::High),
            // wait_for_low
            PinTx::get(PinState::Low),
        ]);

        let mut pin = PinMock::new(&expect);
        pin.set_high().unwrap();

        let delay_transactions = vec![DelayTx::delay_us(1), DelayTx::delay_us(1)];
        let mut delay = CheckedDelay::new(&delay_transactions);

        let mut dht = Dht22::new(pin.clone(), &mut delay);
        dht.wait_for_high().unwrap();
        dht.wait_for_low().unwrap();

        pin.done();
        delay.done();
    }
}
```
