# Main struct

In this section, we will define the main struct for our driver and implement main I2C communication that will let us interact with the RTC hardware.

```rust
pub struct Ds1307<I2C> {
    i2c: I2C,
}
```

This struct holds the I2C interface that we will use to communicate with the DS1307 chip. The generic I2C type allows our driver to work with any I2C implementation that meets the required traits. We keep the struct simple with just the I2C interface since that's all we need to interact with the hardware.

## Implement the RTC HAL's ErrorType

Next, we will implement the ErrorType trait from RTC HAL to specify what error type our driver will use.

```rust
impl<I2C: embedded_hal::i2c::I2c> rtc_hal::error::ErrorType for Ds1307<I2C> {
    type Error = crate::error::Error<I2C::Error>;
}
```

> This implementation involves multiple traits working together. We use the embedded-hal I2c trait instead of concrete types because we want our driver to work with different microcontrollers - they all implement the I2c trait in their own way.

Let's break this down step by step:

**What we're doing:** We're implementing the ErrorType trait for our Ds1307 struct. This trait is required by RTC HAL to know what kind of errors our driver can produce.

**The generic constraint:** `I2C: embedded_hal::i2c::I2c` means our generic I2C type must implement the embedded-hal's I2c trait. This allows our driver to work with STM32, ESP32, or any other microcontroller's I2C implementation.

**The associated type**: We define `Error = crate::error::Error<I2C::Error>`, which means:

- We use our custom Error type we created earlier
- We wrap whatever error type the I2C implementation uses (I2C::Error)
- This makes our driver compatible with different I2C implementations and their specific error types

Basically, we are telling type system: "When our DS1307 driver encounters an error, it returns our custom Error type. If the error comes from I2C communication, our Error type wraps the underlying I2C error"

## Implement Ds1307 

We will start implementing the DS1307 driver with its core functionality. This implementation block will contain all the methods we need to interact with the RTC chip.

> Note: Any other methods we explain after this in this section all go into the same impl block shown below.

```rust
impl<I2C, E> Ds1307<I2C>
where
    I2C: embedded_hal::i2c::I2c<Error = E>,
    E: core::fmt::Debug,
{
    pub fn new(i2c: I2C) -> Self {
        Self { i2c }
    }

    pub fn release_i2c(self) -> I2C {
        self.i2c
    }
}
```

Let's examine the where clause more closely:

```rust
where
    I2C: embedded_hal::i2c::I2c<Error = E>,
    E: core::fmt::Debug,
```

- `I2C: embedded_hal::i2c::I2c<Error = E>`: This means our generic I2C type must implement the embedded-hal I2c trait, and its associated Error type must be the same as our generic E.

- `E: core::fmt::Debug`: This constraint requires that whatever error type the I2C implementation uses must be debuggable. This is already required by the embedded-hal, and all the microcontroller's HAL implement this for their error type also.

In the new function, we take ownership of the I2C instance rather than a mutable reference, following embedded-hal conventions. If user needs to share the I2C bus with other devices, they will use `embedded-hal-bus` crate's sharing implementation. Since embedded-hal has blanket implementations, our driver will work with these shared bus wrappers. We have used a similar pattern with our Max7219 driver in the previous chapter for the SPI instance; in that demo application, we used ExclusiveDevice to manage SPI bus access.

The `release_i2c` method consumes the DS1307 driver and returns the underlying I2C instance, transferring ownership back to the caller. This is useful when you need to reconfigure the I2C bus or use it with other devices after finishing with the DS1307. However, if you're using `embedded-hal-bus` for sharing the I2C bus between multiple devices, you typically won't need this method since the sharing mechanism handles resource management automatically.
