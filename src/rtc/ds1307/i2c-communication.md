# I2C Communication

The DS1307 chip uses a fixed I2C device address of "0x68". We'll declare this as a constant at the top of our module:

```rust
/// DS1307 I2C device address (fixed)
pub const I2C_ADDR: u8 = 0x68;
```

Next, we'll implement helper methods for I2C communication. These are thin wrappers around embedded-hal's I2C methods with additional validation and DS1307-specific logic.

> NOTE: The first parameter for I2C operations will be the I2C address (0x68 for DS1307), followed by the data payload.

## Write to register

We'll start with write_register, which writes a single byte to any DS1307 register. It takes a Register enum value and the data to write, then performs an I2C write transaction with both the register address and value:

```rust
pub(crate) fn write_register(&mut self, register: Register, value: u8) -> Result<(), Error<E>> {
    self.i2c.write(I2C_ADDR, &[register.addr(), value])?;

    Ok(())
}
```


## Read from register

Next, we'll implement read_register to read a single byte from any DS1307 register using the I2C write-read operation:

```rust
pub(crate) fn read_register(&mut self, register: Register) -> Result<u8, Error<E>> {
    let mut data = [0u8; 1];
    self.i2c
        .write_read(I2C_ADDR, &[register.addr()], &mut data)?;

    Ok(data[0])
}
```

## Read multiple bytes

We'll also implement read_register_bytes to read multiple consecutive bytes from the DS1307 starting at a specific register. This is more efficient than multiple single-byte reads when we need to read several registers in sequence, such as when reading the complete date and time:

```rust
pub(crate) fn read_register_bytes(
    &mut self,
    register: Register,
    buffer: &mut [u8],
) -> Result<(), Error<E>> {
    self.i2c.write_read(I2C_ADDR, &[register.addr()], buffer)?;

    Ok(())
}
```

This method takes a starting register and a mutable buffer, then fills the buffer with consecutive bytes from the DS1307. The DS1307 automatically increments its internal register pointer after each byte read, allowing us to read multiple registers in a single I2C transaction for better performance. In the datetime implementation, we will use this method to read the complete date and time from the seconds register (0x00) through the year register (0x06).

## Read bytes at raw address

We'll implement read_bytes_at_address for cases where we need to read from memory locations that aren't covered by our Register enum, such as NVRAM addresses:

```rust
pub(crate) fn read_bytes_at_address(
    &mut self,
    register_addr: u8,
    buffer: &mut [u8],
) -> Result<(), Error<E>> {
    self.i2c.write_read(I2C_ADDR, &[register_addr], buffer)?;

    Ok(())
}
```

This method is similar to read_register_bytes but accepts a raw u8 address instead of our Register enum. This flexibility is essential for accessing the DS1307's NVRAM region (addresses 0x08-0x3F).

## Write raw bytes

We'll implement write_raw_bytes for direct I2C write operations where we need to send a complete data payload including the register address:

```rust
pub(crate) fn write_raw_bytes(&mut self, data: &[u8]) -> Result<(), Error<E>> {
    self.i2c.write(I2C_ADDR, data)?;

    Ok(())
}
```

This method provides low-level access for bulk operations where the first byte must be the target register address followed by the data bytes. It's essential for atomic operations like setting the complete date/time (all 7 registers in one transaction) as well as writing multiple bytes to NVRAM.

## Set register bits

We'll implement set_register_bits for modifying specific bits in DS1307 registers without affecting other bits:

```rust
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
```

We read the current register value, apply a bitwise `OR` with the mask to set the desired bits. We only write back if the value actually changed. This approach preserves existing bits and minimizes I2C traffic by avoiding unnecessary writes.

For example, to halt the clock, we will call `self.set_register_bits(Register::Seconds, CH_BIT)`. This reads the seconds register (which contains BCD time data), sets only the CH bit to 1 to halt the oscillator, and preserves all the existing time data in the other bits.

Let's say the "seconds" register currently contains 0001_0000 (10 seconds in BCD):

```text
Current value:        0001_0000 (10 seconds, clock running)
CH_BIT mask:          1000_0000 (Clock Halt bit)
                      -----------
Bitwise OR result:    1001_0000 (10 seconds with CH bit set)
```

The method preserves the existing 10 seconds value while setting the CH bit to halt the oscillator. In this case, the CH_BIT was not set, so we will send the write request.

## Clear Register bits

We implement clear_register_bits for clearing specific bits in DS1307 registers without affecting other bits:

```rust
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
```

Just like the set_register_bits function, we read the current register value, but we apply a bitwise "AND" with the inverted mask to clear the desired bits. We only write back if the value actually changed. This approach also helps in preserving existing bits and minimizes I2C traffic by avoiding unnecessary writes.

For example, to start the clock, we will call `self.clear_register_bits(Register::Seconds, CH_BIT)`. This reads the seconds register, clears only the CH bit to 0 to start the oscillator, and preserves all the existing time data in the other bits.

Let's say the "seconds" register currently contains 1001_0000 (10 seconds with clock halted):

```text
CH_BIT mask:          1000_0000 (Clock Halt bit)

Inverted mask (~):    0111_1111 (invert all bits of CH_BIT)
Current value:        1001_0000 (10 seconds, clock halted)
                      -----------
Bitwise AND result:   0001_0000 (10 seconds with CH bit cleared)
```

The method preserves the existing 10 seconds value while clearing the CH bit to start the oscillator. In this case, the CH_BIT was set, so we will send the write request.

## Set output Pin to high

We'll implement set_output_high to configure the SQW/OUT pin to output a high logic level:

```rust
pub fn set_output_high(&mut self) -> Result<(), Error<E>> {
    let current = self.read_register(Register::Control)?;
    let mut new_value = current;

    // Disable square wave and set OUT bit high
    new_value &= !SQWE_BIT;
    new_value |= OUT_BIT;

    if new_value != current {
        self.write_register(Register::Control, new_value)
    } else {
        Ok(())
    }
}
```

We first read the current value of the `Control` register and put it into the new_value variable. Then we perform two operations:

1. We apply the bitwise AND with the inverted mask of SQWE_BIT to disable the square wave output
2. Then, we apply the bitwise OR with the OUT_BIT mask to drive the pin high

When square wave generation is disabled (SQWE = 0), the DS1307 uses the OUT bit to control the static output level on the SQW/OUT pin. Setting OUT = 1 drives the pin to a high logic level (typically 3.3V or 5V depending on VCC). The method only writes to the register if the value actually changes, optimizing I2C bus usage.

Let's say the Control register currently contains 0001_0001 (square wave enabled at 4.096 kHz with RS1=0, RS0=1):

```text
Initial value:           0001_0001 (OUT=0, SQWE=1, RS1=0, RS0=1)

Step 1 - Clear SQWE (bit 4):
SQWE_BIT mask:           0001_0000
Inverted mask (~):       1110_1111 (invert all bits of SQWE_BIT)
Current value:           0001_0001
                         -----------
AND result:              0000_0001 (SQWE disabled, RS bits preserved)

Step 2 - Set OUT bit (bit 7):
OUT_BIT mask:            1000_0000
Current value:           0000_0001
                         -----------
OR result:               1000_0001 (OUT=1, static high output)
```

The method preserves other bits (like rate select bits) while configuring the pin for static high output. Since the value changed from 0001_0001 to 1000_0001, we will send the write request.

## Set output Pin to Low

The logic is the same as the set_output_high method except we clear the output bit:

```rust
pub fn set_output_low(&mut self) -> Result<(), Error<E>> {
    let current = self.read_register(Register::Control)?;
    let mut new_value = current;

    // Disable square wave and set OUT bit low
    new_value &= !SQWE_BIT;
    new_value &= !OUT_BIT;

    if new_value != current {
        self.write_register(Register::Control, new_value)
    } else {
        Ok(())
    }
}
```

## NVRAM bounds validation

We'll implement a helper function validate_nvram_bounds to ensure safe access to the DS1307's NVRAM region before performing read or write operations:

```rust
pub(crate) fn validate_nvram_bounds(&self, offset: u8, len: usize) -> Result<(), Error<E>> {
    // Check if offset is within bounds
    if offset >= NVRAM_SIZE {
        return Err(Error::NvramOutOfBounds);
    }

    // Check if remaining space is sufficient
    let remaining_space = NVRAM_SIZE - offset;
    if len > remaining_space as usize {
        return Err(Error::NvramOutOfBounds);
    }

    Ok(())
}
```

This helper function performs two critical validation checks before any NVRAM operation:

- Offset validation: Ensures the starting offset is within the valid NVRAM range (0 to NVRAM_SIZE-1)

- Length validation: Calculates the remaining space from the offset and ensures the requested operation length doesn't exceed the available NVRAM boundary

The DS1307 provides 56 bytes of battery-backed NVRAM (addresses 0x08 to 0x3F), so NVRAM_SIZE would be 56. This function will be used by the NVRAM trait implementation methods like read_nvram() and write_nvram() to ensure all operations stay within the valid memory bounds before performing actual I2C transactions.

## The fullcode for the Ds1307 module (ds1307.rs)

```rust
//! # DS1307 Real-Time Clock Driver

use crate::{
    error::Error,
    registers::{NVRAM_SIZE, OUT_BIT, Register, SQWE_BIT},
};

/// DS1307 I2C device address (fixed)
pub const I2C_ADDR: u8 = 0x68;

/// DS1307 Real-Time Clock driver
pub struct Ds1307<I2C> {
    i2c: I2C,
}

impl<I2C: embedded_hal::i2c::I2c> rtc_hal::error::ErrorType for Ds1307<I2C> {
    type Error = crate::error::Error<I2C::Error>;
}

impl<I2C, E> Ds1307<I2C>
where
    I2C: embedded_hal::i2c::I2c<Error = E>,
    E: core::fmt::Debug,
{
    /// Create a new DS1307 driver instance
    ///
    /// # Parameters
    /// * `i2c` - I2C peripheral that implements the embedded-hal I2c trait
    ///
    /// # Returns
    /// New DS1307 driver instance
    pub fn new(i2c: I2C) -> Self {
        Self { i2c }
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

    /// Write a single byte to a DS1307 register
    pub(crate) fn write_register(&mut self, register: Register, value: u8) -> Result<(), Error<E>> {
        self.i2c.write(I2C_ADDR, &[register.addr(), value])?;

        Ok(())
    }

    /// Read a single byte from a DS1307 register
    pub(crate) fn read_register(&mut self, register: Register) -> Result<u8, Error<E>> {
        let mut data = [0u8; 1];
        self.i2c
            .write_read(I2C_ADDR, &[register.addr()], &mut data)?;

        Ok(data[0])
    }

    /// Read multiple bytes from DS1307 starting at a register
    pub(crate) fn read_register_bytes(
        &mut self,
        register: Register,
        buffer: &mut [u8],
    ) -> Result<(), Error<E>> {
        self.i2c.write_read(I2C_ADDR, &[register.addr()], buffer)?;

        Ok(())
    }

    /// Read multiple bytes from DS1307 starting at a raw address
    pub(crate) fn read_bytes_at_address(
        &mut self,
        register_addr: u8,
        buffer: &mut [u8],
    ) -> Result<(), Error<E>> {
        self.i2c.write_read(I2C_ADDR, &[register_addr], buffer)?;

        Ok(())
    }

    /// Write raw bytes directly to DS1307 via I2C (register address must be first byte)
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
    /// - `register`: The DS1307 register to modify
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
    /// - `register`: The DS1307 register to modify
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

    /// Set the output pin to a static high state
    pub fn set_output_high(&mut self) -> Result<(), Error<E>> {
        let current = self.read_register(Register::Control)?;
        let mut new_value = current;

        // Disable square wave and set OUT bit high
        new_value &= !SQWE_BIT;
        new_value |= OUT_BIT;

        if new_value != current {
            self.write_register(Register::Control, new_value)
        } else {
            Ok(())
        }
    }

    /// Set the output pin to a static low state
    pub fn set_output_low(&mut self) -> Result<(), Error<E>> {
        let current = self.read_register(Register::Control)?;
        let mut new_value = current;

        // Disable square wave and set OUT bit low
        new_value &= !SQWE_BIT;
        new_value &= !OUT_BIT;

        if new_value != current {
            self.write_register(Register::Control, new_value)
        } else {
            Ok(())
        }
    }

    /// Validate NVRAM offset and length parameters before accessing memory.
    ///
    /// Returns an error if:
    /// - The starting offset is outside the available NVRAM range
    /// - The requested length goes beyond the end of NVRAM
    pub(crate) fn validate_nvram_bounds(&self, offset: u8, len: usize) -> Result<(), Error<E>> {
        // Check if offset is within bounds
        if offset >= NVRAM_SIZE {
            return Err(Error::NvramOutOfBounds);
        }

        // Check if remaining space is sufficient
        let remaining_space = NVRAM_SIZE - offset;
        if len > remaining_space as usize {
            return Err(Error::NvramOutOfBounds);
        }

        Ok(())
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::registers::{OUT_BIT, Register, SQWE_BIT};
    use embedded_hal_mock::eh1::i2c::{Mock as I2cMock, Transaction as I2cTransaction};

    const DS1307_ADDR: u8 = 0x68;

    #[test]
    fn test_new() {
        let i2c_mock = I2cMock::new(&[]);
        let ds1307 = Ds1307::new(i2c_mock);
        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_release_i2c() {
        let i2c_mock = I2cMock::new(&[]);
        let ds1307 = Ds1307::new(i2c_mock);

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_write_register() {
        let expectations = vec![I2cTransaction::write(
            DS1307_ADDR,
            vec![Register::Control.addr(), 0x42],
        )];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.write_register(Register::Control, 0x42);
        assert!(result.is_ok());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_write_register_error() {
        let expectations = vec![
            I2cTransaction::write(DS1307_ADDR, vec![Register::Control.addr(), 0x42])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.write_register(Register::Control, 0x42);
        assert!(result.is_err());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_read_register() {
        let expectations = vec![I2cTransaction::write_read(
            DS1307_ADDR,
            vec![Register::Control.addr()],
            vec![0x55],
        )];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.read_register(Register::Control);
        assert_eq!(result.unwrap(), 0x55);

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_read_register_error() {
        let expectations = vec![
            I2cTransaction::write_read(DS1307_ADDR, vec![Register::Control.addr()], vec![0x00])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.read_register(Register::Control);
        assert!(result.is_err());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_read_register_bytes() {
        let expectations = vec![I2cTransaction::write_read(
            DS1307_ADDR,
            vec![Register::Seconds.addr()],
            vec![0x11, 0x22, 0x33],
        )];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let mut buffer = [0u8; 3];
        let result = ds1307.read_register_bytes(Register::Seconds, &mut buffer);
        assert!(result.is_ok());
        assert_eq!(buffer, [0x11, 0x22, 0x33]);

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_read_register_bytes_error() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS1307_ADDR,
                vec![Register::Seconds.addr()],
                vec![0x00, 0x00],
            )
            .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let mut buffer = [0u8; 2];
        let result = ds1307.read_register_bytes(Register::Seconds, &mut buffer);
        assert!(result.is_err());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_read_bytes_at_address() {
        let expectations = vec![I2cTransaction::write_read(
            DS1307_ADDR,
            vec![0x08], // Raw address
            vec![0xAA, 0xBB],
        )];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let mut buffer = [0u8; 2];
        let result = ds1307.read_bytes_at_address(0x08, &mut buffer);
        assert!(result.is_ok());
        assert_eq!(buffer, [0xAA, 0xBB]);

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_read_bytes_at_address_error() {
        let expectations = vec![
            I2cTransaction::write_read(DS1307_ADDR, vec![0x08], vec![0x00])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let mut buffer = [0u8; 1];
        let result = ds1307.read_bytes_at_address(0x08, &mut buffer);
        assert!(result.is_err());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_write_raw_bytes() {
        let expectations = vec![I2cTransaction::write(DS1307_ADDR, vec![0x0E, 0x1C, 0x00])];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.write_raw_bytes(&[0x0E, 0x1C, 0x00]);
        assert!(result.is_ok());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_write_raw_bytes_error() {
        let expectations = vec![
            I2cTransaction::write(DS1307_ADDR, vec![0x0E, 0x1C])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.write_raw_bytes(&[0x0E, 0x1C]);
        assert!(result.is_err());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_set_register_bits_change_needed() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS1307_ADDR,
                vec![Register::Control.addr()],
                vec![0b0000_1000],
            ),
            I2cTransaction::write(DS1307_ADDR, vec![Register::Control.addr(), 0b0001_1000]),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.set_register_bits(Register::Control, 0b0001_0000);
        assert!(result.is_ok());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_set_register_bits_no_change_needed() {
        let expectations = vec![I2cTransaction::write_read(
            DS1307_ADDR,
            vec![Register::Control.addr()],
            vec![0b0001_1000],
        )];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.set_register_bits(Register::Control, 0b0001_0000);
        assert!(result.is_ok());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_set_register_bits_multiple_bits() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS1307_ADDR,
                vec![Register::Control.addr()],
                vec![0b0000_0000],
            ),
            I2cTransaction::write(DS1307_ADDR, vec![Register::Control.addr(), 0b1010_0101]),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.set_register_bits(Register::Control, 0b1010_0101);
        assert!(result.is_ok());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_set_register_bits_read_error() {
        let expectations = vec![
            I2cTransaction::write_read(DS1307_ADDR, vec![Register::Control.addr()], vec![0x00])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.set_register_bits(Register::Control, 0b0001_0000);
        assert!(result.is_err());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_set_register_bits_write_error() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS1307_ADDR,
                vec![Register::Control.addr()],
                vec![0b0000_0000],
            ),
            I2cTransaction::write(DS1307_ADDR, vec![Register::Control.addr(), 0b0001_0000])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.set_register_bits(Register::Control, 0b0001_0000);
        assert!(result.is_err());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_clear_register_bits_change_needed() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS1307_ADDR,
                vec![Register::Control.addr()],
                vec![0b1111_1111],
            ),
            I2cTransaction::write(DS1307_ADDR, vec![Register::Control.addr(), 0b1110_1111]),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.clear_register_bits(Register::Control, 0b0001_0000);
        assert!(result.is_ok());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_clear_register_bits_no_change_needed() {
        let expectations = vec![I2cTransaction::write_read(
            DS1307_ADDR,
            vec![Register::Control.addr()],
            vec![0b1110_1111],
        )];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.clear_register_bits(Register::Control, 0b0001_0000);
        assert!(result.is_ok());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_clear_register_bits_multiple_bits() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS1307_ADDR,
                vec![Register::Control.addr()],
                vec![0b1111_1111],
            ),
            I2cTransaction::write(DS1307_ADDR, vec![Register::Control.addr(), 0b0101_1010]),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.clear_register_bits(Register::Control, 0b1010_0101);
        assert!(result.is_ok());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_clear_register_bits_read_error() {
        let expectations = vec![
            I2cTransaction::write_read(DS1307_ADDR, vec![Register::Control.addr()], vec![0x00])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.clear_register_bits(Register::Control, 0b0001_0000);
        assert!(result.is_err());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_clear_register_bits_write_error() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS1307_ADDR,
                vec![Register::Control.addr()],
                vec![0b1111_1111],
            ),
            I2cTransaction::write(DS1307_ADDR, vec![Register::Control.addr(), 0b1110_1111])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.clear_register_bits(Register::Control, 0b0001_0000);
        assert!(result.is_err());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_set_register_bits_preserves_other_bits() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS1307_ADDR,
                vec![Register::Control.addr()],
                vec![0b1000_0010],
            ),
            I2cTransaction::write(DS1307_ADDR, vec![Register::Control.addr(), 0b1001_0010]),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.set_register_bits(Register::Control, 0b0001_0000);
        assert!(result.is_ok());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_clear_register_bits_preserves_other_bits() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS1307_ADDR,
                vec![Register::Control.addr()],
                vec![0b1001_0010],
            ),
            I2cTransaction::write(DS1307_ADDR, vec![Register::Control.addr(), 0b1000_0010]),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.clear_register_bits(Register::Control, 0b0001_0000);
        assert!(result.is_ok());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_set_output_high_from_sqwe_disabled() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS1307_ADDR,
                vec![Register::Control.addr()],
                vec![0b0000_0000], // SQWE=0, OUT=0
            ),
            I2cTransaction::write(
                DS1307_ADDR,
                vec![Register::Control.addr(), OUT_BIT], // SQWE=0, OUT=1
            ),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.set_output_high();
        assert!(result.is_ok());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_set_output_high_from_sqwe_enabled() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS1307_ADDR,
                vec![Register::Control.addr()],
                vec![SQWE_BIT], // SQWE=1, OUT=0
            ),
            I2cTransaction::write(
                DS1307_ADDR,
                vec![Register::Control.addr(), OUT_BIT], // SQWE=0, OUT=1
            ),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.set_output_high();
        assert!(result.is_ok());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_set_output_high_already_high() {
        let expectations = vec![I2cTransaction::write_read(
            DS1307_ADDR,
            vec![Register::Control.addr()],
            vec![OUT_BIT], // SQWE=0, OUT=1 (already correct state)
        )];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.set_output_high();
        assert!(result.is_ok());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_set_output_low_from_sqwe_disabled() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS1307_ADDR,
                vec![Register::Control.addr()],
                vec![OUT_BIT], // SQWE=0, OUT=1
            ),
            I2cTransaction::write(
                DS1307_ADDR,
                vec![Register::Control.addr(), 0b0000_0000], // SQWE=0, OUT=0
            ),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.set_output_low();
        assert!(result.is_ok());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_set_output_low_from_sqwe_enabled() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS1307_ADDR,
                vec![Register::Control.addr()],
                vec![SQWE_BIT | OUT_BIT], // SQWE=1, OUT=1
            ),
            I2cTransaction::write(
                DS1307_ADDR,
                vec![Register::Control.addr(), 0b0000_0000], // SQWE=0, OUT=0
            ),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.set_output_low();
        assert!(result.is_ok());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_set_output_low_already_low() {
        let expectations = vec![I2cTransaction::write_read(
            DS1307_ADDR,
            vec![Register::Control.addr()],
            vec![0b0000_0000], // SQWE=0, OUT=0 (already correct state)
        )];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.set_output_low();
        assert!(result.is_ok());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_set_output_high_read_error() {
        let expectations = vec![
            I2cTransaction::write_read(DS1307_ADDR, vec![Register::Control.addr()], vec![0x00])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.set_output_high();
        assert!(result.is_err());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_set_output_high_write_error() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS1307_ADDR,
                vec![Register::Control.addr()],
                vec![0b0000_0000],
            ),
            I2cTransaction::write(DS1307_ADDR, vec![Register::Control.addr(), OUT_BIT])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.set_output_high();
        assert!(result.is_err());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_set_output_low_read_error() {
        let expectations = vec![
            I2cTransaction::write_read(DS1307_ADDR, vec![Register::Control.addr()], vec![0x00])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.set_output_low();
        assert!(result.is_err());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_set_output_low_write_error() {
        let expectations = vec![
            I2cTransaction::write_read(DS1307_ADDR, vec![Register::Control.addr()], vec![OUT_BIT]),
            I2cTransaction::write(DS1307_ADDR, vec![Register::Control.addr(), 0b0000_0000])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.set_output_low();
        assert!(result.is_err());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_output_functions_preserve_other_bits() {
        // Test that output functions preserve other control register bits
        let other_bits = 0b1100_0000; // Some other bits set

        let expectations = vec![
            I2cTransaction::write_read(
                DS1307_ADDR,
                vec![Register::Control.addr()],
                vec![other_bits | SQWE_BIT], // SQWE enabled with other bits
            ),
            I2cTransaction::write(
                DS1307_ADDR,
                vec![Register::Control.addr(), other_bits | OUT_BIT], // SQWE disabled, OUT high, other bits preserved
            ),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.set_output_high();
        assert!(result.is_ok());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }
}
```
