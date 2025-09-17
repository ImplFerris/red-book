# I2C Communication

The DS3231 chip also use the same fixed I2C device address of "0x68". We'll declare this as a constant at the top of our module:

```rust
/// DS3231 I2C device address (fixed)
pub const I2C_ADDR: u8 = 0x68;
```

## The fullcode for the Ds3231 module (ds3231.rs)

The remaining functions for reading and writing registers, and setting and clearing register bits are identical to the DS1307.

```rust
//! DS3231 Real-Time Clock Driver

use embedded_hal::i2c::I2c;

use crate::{error::Error, registers::Register};

/// DS3231 I2C device address (fixed)
pub const I2C_ADDR: u8 = 0x68;

/// Default base century for year calculations (2000-2099).
/// Since the DS3231's century bit meaning is ambiguous, we assume
/// years 00-99 represent the 21st century by default.
pub const DEFAULT_BASE_CENTURY: u8 = 20;

/// DS3231 Real-Time Clock driver
pub struct Ds3231<I2C> {
    i2c: I2C,
    pub(crate) base_century: u8,
}

impl<I2C: embedded_hal::i2c::I2c> rtc_hal::error::ErrorType for Ds3231<I2C> {
    type Error = crate::error::Error<I2C::Error>;
}

impl<I2C, E> Ds3231<I2C>
where
    I2C: I2c<Error = E>,
    E: core::fmt::Debug,
{
    /// Create a new DS3231 driver instance
    ///
    /// # Parameters
    /// * `i2c` - I2C peripheral that implements the embedded-hal I2c trait
    ///
    /// # Returns
    /// New DS3231 driver instance
    pub fn new(i2c: I2C) -> Self {
        Self {
            i2c,
            base_century: DEFAULT_BASE_CENTURY,
        }
    }

    /// Sets the base century for year calculations.
    ///
    /// The DS3231 stores years as 00-99 in BCD format. This base century
    /// determines how those 2-digit years are interpreted as full 4-digit years.
    ///
    /// # Arguments
    ///
    /// * `base_century` - The century to use (e.g., 20 for 2000-2099, 21 for 2100-2199)
    ///
    /// # Returns
    ///
    /// Returns `Err(Error::InvalidBaseCentury)` if base_century is less than 19.
    ///
    /// # Examples
    ///
    /// ```
    /// // Years 00-99 will be interpreted as 2000-2099
    /// let rtc = Ds3231::new(i2c).with_base_century(20)?;
    ///
    /// // Years 00-99 will be interpreted as 2100-2199
    /// let rtc = Ds3231::new(i2c).with_base_century(21)?;
    /// ```
    pub fn set_base_century(&mut self, base_century: u8) -> Result<(), Error<E>> {
        if base_century < 19 {
            return Err(Error::InvalidBaseCentury);
        }
        self.base_century = base_century;
        Ok(())
    }

    /// Returns the underlying I2C bus instance, consuming the driver.
    ///
    /// This allows the user to reuse the I2C bus for other purposes
    /// after the driver is no longer needed.
    ///
    /// However, if you are using [`embedded-hal-bus`](https://crates.io/crates/embedded-hal-bus),
    /// you typically do not need `release_i2c`.
    /// In that case the crate takes care of the sharing
    pub fn release_i2c(self) -> I2C {
        self.i2c
    }

    /// Write a single byte to a DS3231 register
    pub(crate) fn write_register(&mut self, register: Register, value: u8) -> Result<(), Error<E>> {
        self.i2c.write(I2C_ADDR, &[register.addr(), value])?;

        Ok(())
    }

    /// Read a single byte from a DS3231 register
    pub(crate) fn read_register(&mut self, register: Register) -> Result<u8, Error<E>> {
        let mut data = [0u8; 1];
        self.i2c
            .write_read(I2C_ADDR, &[register.addr()], &mut data)?;

        Ok(data[0])
    }

    /// Read multiple bytes from DS3231 starting at a register
    pub(crate) fn read_register_bytes(
        &mut self,
        register: Register,
        buffer: &mut [u8],
    ) -> Result<(), Error<E>> {
        self.i2c.write_read(I2C_ADDR, &[register.addr()], buffer)?;

        Ok(())
    }

    // Read multiple bytes from DS3231 starting at a raw address
    // pub(crate) fn read_bytes_at_address(
    //     &mut self,
    //     register_addr: u8,
    //     buffer: &mut [u8],
    // ) -> Result<(), Error<E>> {
    //     self.i2c.write_read(I2C_ADDR, &[register_addr], buffer)?;

    //     Ok(())
    // }

    /// Write raw bytes directly to DS3231 via I2C (register address must be first byte)
    pub(crate) fn write_raw_bytes(&mut self, data: &[u8]) -> Result<(), Error<E>> {
        self.i2c.write(I2C_ADDR, data)?;

        Ok(())
    }

    /// Read-modify-write operation for setting bits
    ///
    /// Performs a read-modify-write operation to set the bits specified by the mask
    /// while preserving all other bits in the register. Only performs a write if
    /// the register value would actually change, optimizing I2C bus usage.
    ///
    /// # Parameters
    /// - `register`: The DS3231 register to modify
    /// - `mask`: Bit mask where `1` bits will be set, `0` bits will be ignored
    ///
    /// # Example
    /// ```ignore
    /// // Set bits 2 and 4 in the control register
    /// self.set_register_bits(Register::Control, 0b0001_0100)?;
    /// ```
    ///
    /// # I2C Operations
    /// - 1 read + 1 write (if change needed)
    /// - 1 read only (if no change needed)
    pub(crate) fn set_register_bits(
        &mut self,
        register: Register,
        mask: u8,
    ) -> Result<(), Error<E>> {
        let current = self.read_register(register)?;
        let new_value = current | mask;
        if new_value != current {
            self.write_register(register, new_value)
        } else {
            Ok(())
        }
    }

    /// Read-modify-write operation for clearing bits
    ///
    /// Performs a read-modify-write operation to clear the bits specified by the mask
    /// while preserving all other bits in the register. Only performs a write if
    /// the register value would actually change, optimizing I2C bus usage.
    ///
    /// # Parameters
    /// - `register`: The DS3231 register to modify
    /// - `mask`: Bit mask where `1` bits will be cleared, `0` bits will be ignored
    ///
    /// # Example
    /// ```ignore
    /// // Clear the Clock Halt bit (bit 7) in seconds register
    /// self.clear_register_bits(Register::Seconds, 0b1000_0000)?;
    /// ```
    ///
    /// # I2C Operations
    /// - 1 read + 1 write (if change needed)
    /// - 1 read only (if no change needed)
    pub(crate) fn clear_register_bits(
        &mut self,
        register: Register,
        mask: u8,
    ) -> Result<(), Error<E>> {
        let current = self.read_register(register)?;
        let new_value = current & !mask;
        if new_value != current {
            self.write_register(register, new_value)
        } else {
            Ok(())
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::error::Error;
    use crate::registers::Register;
    use embedded_hal_mock::eh1::i2c::{Mock as I2cMock, Transaction as I2cTransaction};

    const DS3231_ADDR: u8 = 0x68;

    #[test]
    fn test_new() {
        let i2c_mock = I2cMock::new(&[]);
        let ds3231 = Ds3231::new(i2c_mock);

        assert_eq!(ds3231.base_century, DEFAULT_BASE_CENTURY);

        let mut i2c_mock = ds3231.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_set_base_century_valid() {
        let i2c_mock = I2cMock::new(&[]);
        let mut ds3231 = Ds3231::new(i2c_mock);

        let result = ds3231.set_base_century(21);
        assert!(result.is_ok());
        assert_eq!(ds3231.base_century, 21);

        let mut i2c_mock = ds3231.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_set_base_century_minimum_valid() {
        let i2c_mock = I2cMock::new(&[]);
        let mut ds3231 = Ds3231::new(i2c_mock);

        let result = ds3231.set_base_century(19);
        assert!(result.is_ok());
        assert_eq!(ds3231.base_century, 19);

        let mut i2c_mock = ds3231.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_set_base_century_invalid() {
        let i2c_mock = I2cMock::new(&[]);
        let mut ds3231 = Ds3231::new(i2c_mock);

        let result = ds3231.set_base_century(18);
        assert!(matches!(result, Err(Error::InvalidBaseCentury)));
        assert_eq!(ds3231.base_century, DEFAULT_BASE_CENTURY);

        let mut i2c_mock = ds3231.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_write_register() {
        let expectations = vec![I2cTransaction::write(
            DS3231_ADDR,
            vec![Register::Control.addr(), 0x42],
        )];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(i2c_mock);

        let result = ds3231.write_register(Register::Control, 0x42);
        assert!(result.is_ok());

        let mut i2c_mock = ds3231.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_write_register_error() {
        let expectations = vec![
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), 0x42])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(i2c_mock);

        let result = ds3231.write_register(Register::Control, 0x42);
        assert!(result.is_err());

        let mut i2c_mock = ds3231.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_read_register() {
        let expectations = vec![I2cTransaction::write_read(
            DS3231_ADDR,
            vec![Register::Control.addr()],
            vec![0x55],
        )];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(i2c_mock);

        let result = ds3231.read_register(Register::Control);
        assert_eq!(result.unwrap(), 0x55);

        let mut i2c_mock = ds3231.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_read_register_error() {
        let expectations = vec![
            I2cTransaction::write_read(DS3231_ADDR, vec![Register::Control.addr()], vec![0x00])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(i2c_mock);

        let result = ds3231.read_register(Register::Control);
        assert!(result.is_err());

        let mut i2c_mock = ds3231.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_read_register_bytes() {
        let expectations = vec![I2cTransaction::write_read(
            DS3231_ADDR,
            vec![Register::Seconds.addr()],
            vec![0x11, 0x22, 0x33],
        )];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(i2c_mock);

        let mut buffer = [0u8; 3];
        let result = ds3231.read_register_bytes(Register::Seconds, &mut buffer);
        assert!(result.is_ok());
        assert_eq!(buffer, [0x11, 0x22, 0x33]);

        let mut i2c_mock = ds3231.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_read_register_bytes_error() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Seconds.addr()],
                vec![0x00, 0x00],
            )
            .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(i2c_mock);

        let mut buffer = [0u8; 2];
        let result = ds3231.read_register_bytes(Register::Seconds, &mut buffer);
        assert!(result.is_err());

        let mut i2c_mock = ds3231.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_write_raw_bytes() {
        let expectations = vec![I2cTransaction::write(DS3231_ADDR, vec![0x0E, 0x1C, 0x00])];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(i2c_mock);

        let result = ds3231.write_raw_bytes(&[0x0E, 0x1C, 0x00]);
        assert!(result.is_ok());

        let mut i2c_mock = ds3231.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_write_raw_bytes_error() {
        let expectations = vec![
            I2cTransaction::write(DS3231_ADDR, vec![0x0E, 0x1C])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(i2c_mock);

        let result = ds3231.write_raw_bytes(&[0x0E, 0x1C]);
        assert!(result.is_err());

        let mut i2c_mock = ds3231.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_set_register_bits_change_needed() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b0000_1000],
            ),
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), 0b0001_1000]),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(i2c_mock);

        let result = ds3231.set_register_bits(Register::Control, 0b0001_0000);
        assert!(result.is_ok());

        let mut i2c_mock = ds3231.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_set_register_bits_no_change_needed() {
        let expectations = vec![I2cTransaction::write_read(
            DS3231_ADDR,
            vec![Register::Control.addr()],
            vec![0b0001_1000],
        )];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(i2c_mock);

        let result = ds3231.set_register_bits(Register::Control, 0b0001_0000);
        assert!(result.is_ok());

        let mut i2c_mock = ds3231.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_set_register_bits_multiple_bits() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b0000_0000],
            ),
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), 0b1010_0101]),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(i2c_mock);

        let result = ds3231.set_register_bits(Register::Control, 0b1010_0101);
        assert!(result.is_ok());

        let mut i2c_mock = ds3231.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_set_register_bits_read_error() {
        let expectations = vec![
            I2cTransaction::write_read(DS3231_ADDR, vec![Register::Control.addr()], vec![0x00])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(i2c_mock);

        let result = ds3231.set_register_bits(Register::Control, 0b0001_0000);
        assert!(result.is_err());

        let mut i2c_mock = ds3231.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_set_register_bits_write_error() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b0000_0000],
            ),
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), 0b0001_0000])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(i2c_mock);

        let result = ds3231.set_register_bits(Register::Control, 0b0001_0000);
        assert!(result.is_err());

        let mut i2c_mock = ds3231.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_clear_register_bits_change_needed() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b1111_1111],
            ),
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), 0b1110_1111]),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(i2c_mock);

        let result = ds3231.clear_register_bits(Register::Control, 0b0001_0000);
        assert!(result.is_ok());

        let mut i2c_mock = ds3231.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_clear_register_bits_no_change_needed() {
        let expectations = vec![I2cTransaction::write_read(
            DS3231_ADDR,
            vec![Register::Control.addr()],
            vec![0b1110_1111],
        )];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(i2c_mock);

        let result = ds3231.clear_register_bits(Register::Control, 0b0001_0000);
        assert!(result.is_ok());

        let mut i2c_mock = ds3231.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_clear_register_bits_multiple_bits() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b1111_1111],
            ),
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), 0b0101_1010]),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(i2c_mock);

        let result = ds3231.clear_register_bits(Register::Control, 0b1010_0101);
        assert!(result.is_ok());

        let mut i2c_mock = ds3231.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_clear_register_bits_read_error() {
        let expectations = vec![
            I2cTransaction::write_read(DS3231_ADDR, vec![Register::Control.addr()], vec![0x00])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(i2c_mock);

        let result = ds3231.clear_register_bits(Register::Control, 0b0001_0000);
        assert!(result.is_err());

        let mut i2c_mock = ds3231.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_clear_register_bits_write_error() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b1111_1111],
            ),
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), 0b1110_1111])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(i2c_mock);

        let result = ds3231.clear_register_bits(Register::Control, 0b0001_0000);
        assert!(result.is_err());

        let mut i2c_mock = ds3231.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_constants() {
        assert_eq!(I2C_ADDR, 0x68);
        assert_eq!(DEFAULT_BASE_CENTURY, 20);
    }

    #[test]
    fn test_set_register_bits_preserves_other_bits() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b1000_0010],
            ),
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), 0b1001_0010]),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(i2c_mock);

        let result = ds3231.set_register_bits(Register::Control, 0b0001_0000);
        assert!(result.is_ok());

        let mut i2c_mock = ds3231.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_clear_register_bits_preserves_other_bits() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b1001_0010],
            ),
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), 0b1000_0010]),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(i2c_mock);

        let result = ds3231.clear_register_bits(Register::Control, 0b0001_0000);
        assert!(result.is_ok());

        let mut i2c_mock = ds3231.release_i2c();
        i2c_mock.done();
    }
}
```
