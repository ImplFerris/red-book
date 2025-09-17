# Main struct

In this section, we will define the main struct for our driver and implement main I2C communication that will let us interact with the RTC hardware.

```rust
pub const DEFAULT_BASE_CENTURY: u8 = 20;

pub struct Ds3231<I2C> {
    i2c: I2C,
    pub(crate) base_century: u8,
}
```

Unlike the DS1307, this driver struct has one more field. The base_century field lets users set which century the year represents. For example, if users want the year 25 to mean 2125 instead of 2025, they can change the base century.

## Implement the RTC HAL's ErrorType

Next, we will implement the ErrorType trait from RTC HAL to specify what error type our driver will use.

```rust
impl<I2C: embedded_hal::i2c::I2c> rtc_hal::error::ErrorType for Ds3231<I2C> {
    type Error = crate::error::Error<I2C::Error>;
}
```

## Implement Ds3231

We will start implementing the DS3231 driver with its core functionality. This implementation block will contain all the methods we need to interact with the RTC chip.

> Note: Any other methods we explain after this in this section all go into the same impl block shown below.

```rust
impl<I2C, E> Ds3231<I2C>
where
    I2C: embedded_hal::i2c::I2c<Error = E>,
    E: core::fmt::Debug,
{
    pub fn new(i2c: I2C) -> Self {
        Self {
            i2c,
            base_century: DEFAULT_BASE_CENTURY,
        }
    }

    pub fn set_base_century(&mut self, base_century: u8) -> Result<(), Error<E>> {
        if base_century < 19 {
            return Err(Error::InvalidBaseCentury);
        }
        self.base_century = base_century;
        Ok(())
    }

    pub fn release_i2c(self) -> I2C {
        self.i2c
    }
}
```

We use a default base century of 20 and provide a set_base_century function in case users want to change the base century.
