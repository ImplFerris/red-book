# Timing Utilities for Pin State Changes

Before we implement the DHT22 communication protocol, we need a few utility functions to handle precise timing and pin state monitoring. These functions help us wait for the data line to go high or low, with a timeout to avoid getting stuck if the sensor becomes unresponsive.

The TIMEOUT_US constant defines the maximum number of microseconds to wait before giving up. It is used by the helper function to avoid waiting forever when the sensor does not switch to the expected pin state.

```rust
const TIMEOUT_US: u8 = 100;
```

All the functions shown below will be part of the Dht22 struct implementation, including the ones we will define in the next chapter.

Next, we define a generic helper that waits for a pin to reach a specific state:

```rust
fn wait_for_state<F>(delay: &mut DELAY, mut condition: F) -> Result<(), DhtError<E>>
where
    F: FnMut() -> Result<bool, E>,
{
    for _ in 0..TIMEOUT_US {
        if condition()? {
            return Ok(());
        }
        delay.delay_us(1);
    }
    Err(DhtError::Timeout)
}
```
This function retries the given condition for up to TIMEOUT_US microseconds. If the condition becomes true within that time, it returns Ok(()). Otherwise, it returns a timeout error. The condition is a Rust closure, which makes it easy to pass different pin checks like is_high() or is_low().

Now we define two simple wrappers that wait for the data line to go high or low:

```rust
fn wait_for_high(&mut self) -> Result<(), DhtError<E>> {
    Self::wait_for_state(&mut self.delay, || self.pin.is_high())
}
```

```rust
fn wait_for_low(&mut self) -> Result<(), DhtError<E>> {
    Self::wait_for_state(&mut self.delay, || self.pin.is_low())
}
```

These functions call wait_for_state and pass the appropriate closure. They wrap the is_high() and is_low() methods and help make the main protocol code more readable.

In simple words, it helps us wait until the pin becomes high or low, but only for a short time. If the pin doesn't change in time, we stop and return an error. This way, the program doesn't hang if the sensor stops responding.
