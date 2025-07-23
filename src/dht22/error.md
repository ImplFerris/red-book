# Error Handling

The DHT22 sensor communication is time-sensitive and error-prone. To deal with possible failures, we define a custom DhtError enum that describes what can go wrong during a sensor read.

Let's create the error module. Add the following line in lib.rs:

```rust
pub mod error;

// re-export the DhtError for library users
pub use error::DhtError;
```

## Contents of error.rs file

Create an error.rs file inside the src folder and add the following code:

```rust
/// Possible errors from the DHT22 driver.
#[derive(Debug, PartialEq, Eq)]
pub enum DhtError<E> {
    /// Timed out waiting for a pin state change.
    Timeout,
    /// Checksum did not match the received data.
    ChecksumMismatch,
    /// Error from the GPIO pin (input/output).
    PinError(E),
}

impl<E> From<E> for DhtError<E> {
    fn from(value: E) -> Self {
        Self::PinError(value)
    }
}
```

The DhtError enum represents three different kinds of errors that can occur during sensor communication.

- **Timeout:** If the sensor does not respond within the expected time during any stage of communication, we return DhtError::Timeout. This can happen, for example, when we wait for the sensor to pull the data line low or high but it never happens.

- **ChecksumMismatch:** After receiving the 5 bytes of data, we calculate the checksum from the first 4 bytes and compare it with the checksum byte sent by the sensor. If they do not match, we return DhtError::ChecksumMismatch. This typically means that some bits were corrupted during transmission.

- **PinError:** Sometimes, GPIO operations themselves can fail. For example, setting a pin high or low, or reading its level, may return an error depending on the HAL used. We wrap such errors using DhtError::PinError so that they can be reported along with the rest. 

We also implement From<E> for DhtError<E>. This allows any pin-level error to be automatically converted into our error type. It enables us to use the ? operator when calling GPIO methods inside the driver, without having to convert the error manually every time.
