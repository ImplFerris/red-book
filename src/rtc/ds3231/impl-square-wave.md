# impl SquareWave

Let's implement the Square Wave trait to enable square wave output support for the DS3231. The core concepts are similar to the DS1307 implementation covered in the previous chapter, but the DS3231 has different frequency options and control mechanisms.

## Helper Function

First, we create a helper function that converts the SquareWaveFreq enum to the correct bit pattern for the RS bits in the DS3231:

```rust
// Note: FYI, this shoudl be outside the impl block
fn freq_to_bits<E>(freq: SquareWaveFreq) -> Result<u8, Error<E>>
where
    E: core::fmt::Debug,
{
    match freq {
        SquareWaveFreq::Hz1 => Ok(0b0000_0000),
        SquareWaveFreq::Hz1024 => Ok(0b0000_1000),
        SquareWaveFreq::Hz4096 => Ok(0b0001_0000),
        SquareWaveFreq::Hz8192 => Ok(0b0001_1000),
        _ => Err(Error::UnsupportedSqwFrequency),
    }
}
```
Note that the DS3231 supports different frequencies than the DS1307 and uses two RS bits (RS2 and RS1) in different bit positions in the control register.

## Key Difference: INTCN Bit Control

Unlike the DS1307 which uses an SQWE bit to enable square wave output, the DS3231 uses the INTCN (Interrupt Control) bit to choose between two modes:

- INTCN = 0: Square wave output mode
- INTCN = 1: Interrupt output mode (default


## Enable the Square Wave Output

In this method, we enable square wave output while preserving the current frequency settings. We simply clear the INTCN bit (bit 2) to switch the INT/SQW pin from interrupt mode to square wave mode. When INTCN = 0, the pin outputs a square wave at the frequency determined by the RS bits.

```rust
fn enable_square_wave(&mut self) -> Result<(), Self::Error> {
    // Clear INTCN bit to enable square wave mode (0 = square wave, 1 = interrupt)
    self.clear_register_bits(Register::Control, INTCN_BIT)
}
```

## Disable the Square Wave Output

In this method, we disable square wave output by setting the INTCN bit to 1, which switches the pin back to interrupt mode. This is the DS3231's default state, where the pin can be used for alarm interrupts instead of square wave generation.

```rust
fn disable_square_wave(&mut self) -> Result<(), Self::Error> {
    // Set INTCN bit to enable interrupt mode (disable square wave)
    self.set_register_bits(Register::Control, INTCN_BIT)
}
```

## Change Frequency

In this method, we change the square wave frequency without affecting the enable/disable state. We use the helper function to convert the frequency enum to RS bits, then update only the RS2 and RS1 bits in the control register while preserving all other settings.

```rust
fn set_square_wave_frequency(&mut self, freq: SquareWaveFreq) -> Result<(), Self::Error> {
    // Convert frequency to RS bits
    let rs_bits = freq_to_bits(freq)?;

    // Read current control register
    let current = self.read_register(Register::Control)?;
    let mut new_value = current;

    // Clear existing RS bits and set new ones
    new_value &= !RS_MASK;
    new_value |= rs_bits; // Set the new RS bits

    // Only write if changed
    if new_value != current {
        self.write_register(Register::Control, new_value)
    } else {
        Ok(())
    }
}
```

## Start Square Wave

In this method, we combine frequency setting and enabling in one operation. We set the desired frequency using the RS bits, then enable square wave output by clearing the INTCN bit. This is equivalent to calling set_square_wave_frequency() followed by enable_square_wave(), but more efficient as it requires only one register write.

```rust
fn start_square_wave(&mut self, freq: SquareWaveFreq) -> Result<(), Self::Error> {
    let rs_bits = freq_to_bits(freq)?;
    let current = self.read_register(Register::Control)?;
    let mut new_value = current;

    // Clear frequency bits and set new ones
    new_value &= !RS_MASK;
    new_value |= rs_bits;

    // Enable square wave
    new_value &= !INTCN_BIT;

    // Only write if changed
    if new_value != current {
        self.write_register(Register::Control, new_value)
    } else {
        Ok(())
    }
}
```

## The full code for the square wave module (square_wave.rs)

```rust
//! DS3231 Square Wave Output Support
//!
//! This module provides an implementation of the [`SquareWave`] trait for the
//! [`Ds3231`] RTC.
//!
//! The DS3231 supports four square wave output frequencies on the INT/SQW pin:
//! 1 Hz, 1.024 kHz, 4.096 kHz, and 8.192 kHz. Other frequencies defined in
//! [`SquareWaveFreq`] will result in an error.
//!
//! Note: The DS3231's dedicated 32 kHz output pin is not controlled by this
//! implementation, only the configurable INT/SQW pin frequencies.

pub use rtc_hal::square_wave::SquareWave;
pub use rtc_hal::square_wave::SquareWaveFreq;

use crate::Ds3231;
use crate::error::Error;
use crate::registers::{INTCN_BIT, RS_MASK, Register};

/// Convert a [`SquareWaveFreq`] into the corresponding Ds3231 RS bits.
///
/// Returns an error if the frequency is not supported by the Ds3231.
fn freq_to_bits<E>(freq: SquareWaveFreq) -> Result<u8, Error<E>>
where
    E: core::fmt::Debug,
{
    match freq {
        SquareWaveFreq::Hz1 => Ok(0b0000_0000),
        SquareWaveFreq::Hz1024 => Ok(0b0000_1000),
        SquareWaveFreq::Hz4096 => Ok(0b0001_0000),
        SquareWaveFreq::Hz8192 => Ok(0b0001_1000),
        _ => Err(Error::UnsupportedSqwFrequency),
    }
}

impl<I2C> SquareWave for Ds3231<I2C>
where
    I2C: embedded_hal::i2c::I2c,
{
    /// Enable the square wave output
    fn enable_square_wave(&mut self) -> Result<(), Self::Error> {
        // Clear INTCN bit to enable square wave mode (0 = square wave, 1 = interrupt)
        self.clear_register_bits(Register::Control, INTCN_BIT)
    }

    /// Disable the square wave output.
    fn disable_square_wave(&mut self) -> Result<(), Self::Error> {
        // Set INTCN bit to enable interrupt mode (disable square wave)
        self.set_register_bits(Register::Control, INTCN_BIT)
    }

    fn set_square_wave_frequency(&mut self, freq: SquareWaveFreq) -> Result<(), Self::Error> {
        // Convert frequency to RS bits
        let rs_bits = freq_to_bits(freq)?;

        // Read current control register
        let current = self.read_register(Register::Control)?;
        let mut new_value = current;

        // Clear existing RS bits and set new ones
        new_value &= !RS_MASK;
        new_value |= rs_bits; // Set the new RS bits

        // Only write if changed
        if new_value != current {
            self.write_register(Register::Control, new_value)
        } else {
            Ok(())
        }
    }

    fn start_square_wave(&mut self, freq: SquareWaveFreq) -> Result<(), Self::Error> {
        let rs_bits = freq_to_bits(freq)?;
        let current = self.read_register(Register::Control)?;
        let mut new_value = current;

        // Clear frequency bits and set new ones
        new_value &= !RS_MASK;
        new_value |= rs_bits;

        // Enable square wave
        new_value &= !INTCN_BIT;

        // Only write if changed
        if new_value != current {
            self.write_register(Register::Control, new_value)
        } else {
            Ok(())
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use embedded_hal_mock::eh1::i2c::{Mock as I2cMock, Transaction as I2cTransaction};
    use rtc_hal::square_wave::{SquareWave, SquareWaveFreq};

    const DS3231_ADDR: u8 = 0x68;

    #[test]
    fn test_freq_to_bits_supported_frequencies() {
        assert_eq!(
            freq_to_bits::<()>(SquareWaveFreq::Hz1).unwrap(),
            0b0000_0000
        );
        assert_eq!(
            freq_to_bits::<()>(SquareWaveFreq::Hz1024).unwrap(),
            0b0000_1000
        );
        assert_eq!(
            freq_to_bits::<()>(SquareWaveFreq::Hz4096).unwrap(),
            0b0001_0000
        );
        assert_eq!(
            freq_to_bits::<()>(SquareWaveFreq::Hz8192).unwrap(),
            0b0001_1000
        );
    }

    #[test]
    fn test_freq_to_bits_unsupported_frequency() {
        let result = freq_to_bits::<()>(SquareWaveFreq::Hz32768);
        assert!(matches!(result, Err(Error::UnsupportedSqwFrequency)));
    }

    #[test]
    fn test_enable_square_wave() {
        let expectations = vec![
            // transaction related to the reading the control register
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b0000_0100],
            ),
            // transaction related to the writing the control register back
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), 0b0000_0000]),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.enable_square_wave();
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_enable_square_wave_already_enabled() {
        let expectations = vec![I2cTransaction::write_read(
            DS3231_ADDR,
            vec![Register::Control.addr()],
            vec![0b0000_0000],
        )];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.enable_square_wave();
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_disable_square_wave() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b0000_0000],
            ),
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), 0b0000_0100]),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.disable_square_wave();
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_set_square_wave_frequency_1hz() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b0001_1000],
            ),
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), 0b0000_0000]),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.set_square_wave_frequency(SquareWaveFreq::Hz1);
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_set_square_wave_frequency_1024hz() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b0000_0000],
            ),
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), 0b0000_1000]),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.set_square_wave_frequency(SquareWaveFreq::Hz1024);
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_set_square_wave_frequency_4096hz() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b0000_1000],
            ),
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), 0b0001_0000]),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.set_square_wave_frequency(SquareWaveFreq::Hz4096);
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_set_square_wave_frequency_8192hz() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b0000_0000],
            ),
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), 0b0001_1000]),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.set_square_wave_frequency(SquareWaveFreq::Hz8192);
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_set_square_wave_frequency_no_change_needed() {
        let expectations = vec![I2cTransaction::write_read(
            DS3231_ADDR,
            vec![Register::Control.addr()],
            vec![0b0001_0000], // The rs bits are for 4.096kHz
        )];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.set_square_wave_frequency(SquareWaveFreq::Hz4096);
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_set_square_wave_frequency_preserves_other_bits() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b1100_0100],
            ),
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), 0b1100_1100]),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.set_square_wave_frequency(SquareWaveFreq::Hz1024);
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_set_square_wave_frequency_unsupported() {
        let expectations = vec![];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.set_square_wave_frequency(SquareWaveFreq::Hz32768);
        assert!(matches!(result, Err(Error::UnsupportedSqwFrequency)));

        i2c_mock.done();
    }

    #[test]
    fn test_start_square_wave_1hz() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b0001_1100],
            ),
            // Sets the bit 2 to 0
            // Set RS1 & RS2 bit value to 0 for 1 Hz
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), 0b0000_0000]),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.start_square_wave(SquareWaveFreq::Hz1);
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_start_square_wave_1024hz() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b1000_0100],
            ),
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), 0b1000_1000]),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.start_square_wave(SquareWaveFreq::Hz1024);
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_start_square_wave_4096hz() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b0100_1100],
            ),
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), 0b0101_0000]),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.start_square_wave(SquareWaveFreq::Hz4096);
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_start_square_wave_8192hz() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b0000_0100],
            ),
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), 0b0001_1000]),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.start_square_wave(SquareWaveFreq::Hz8192);
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_start_square_wave_already_configured() {
        let expectations = vec![I2cTransaction::write_read(
            DS3231_ADDR,
            vec![Register::Control.addr()],
            vec![0b0000_1000],
        )];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.start_square_wave(SquareWaveFreq::Hz1024);
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_start_square_wave_preserves_other_bits() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b1010_0100],
            ),
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), 0b1010_0000]),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.start_square_wave(SquareWaveFreq::Hz1);
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_start_square_wave_unsupported_frequency() {
        let expectations = vec![];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.start_square_wave(SquareWaveFreq::Hz32768);
        assert!(matches!(result, Err(Error::UnsupportedSqwFrequency)));

        i2c_mock.done();
    }

    #[test]
    fn test_i2c_read_error_handling() {
        let expectations = vec![
            I2cTransaction::write_read(DS3231_ADDR, vec![Register::Control.addr()], vec![0x00])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.enable_square_wave();
        assert!(result.is_err());

        i2c_mock.done();
    }

    #[test]
    fn test_i2c_write_error_handling() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b0000_0100],
            ),
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), 0b0000_0000])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.enable_square_wave();
        assert!(result.is_err());

        i2c_mock.done();
    }

    #[test]
    fn test_rs_mask_coverage() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b1111_1111],
            ),
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), 0b1110_1111]),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.set_square_wave_frequency(SquareWaveFreq::Hz1024);
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_intcn_bit_manipulation() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b1111_1111],
            ),
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), 0b1111_1011]),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.enable_square_wave();
        assert!(result.is_ok());

        i2c_mock.done();
    }
}
```
