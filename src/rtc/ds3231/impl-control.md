# impl RtcPowerControl

Now that we have implemented the core Rtc trait, we can extend our DS3231 driver with additional feature-specific traits. Let's start with the `RtcPowerControl` trait, which allows us to halt or start the clock oscillator.

```rust
impl<I2C> RtcPowerControl for Ds3231<I2C>
where
    I2C: embedded_hal::i2c::I2c,
{
    fn start_clock(&mut self) -> Result<(), Self::Error> {
        self.clear_register_bits(Register::Control, EOSC_BIT)
    }

    fn halt_clock(&mut self) -> Result<(), Self::Error> {
        self.set_register_bits(Register::Control, EOSC_BIT)
    }
}
```

Unlike the DS1307 which uses a Clock Halt bit in the Seconds register, the DS3231 uses the EOSC (Enable Oscillator) bit in the Control register. However, there's an important distinction: the DS3231's power control behavior.

**Power Supply Awareness:** The DS3231's EOSC bit only affects operation when running on battery power (VBAT). When the main power supply (VCC) is present, the oscillator always runs regardless of the EOSC bit setting. This ensures reliable timekeeping during normal operation while allowing power conservation during battery backup.

## The Full code for the control module (control.rs)

```rust
//! Power control implementation for the DS3231
//!
//! This module provides power management functionality for the DS3231 RTC chip,
//! implementing the `RtcPowerControl` trait to allow starting and stopping the
//! internal oscillator that drives timekeeping operations.
//!
//! ## Hardware Behavior
//!
//! The DS3231 uses the Enable Oscillator (EOSC) bit in the Control Register (0Eh)
//! to control oscillator operation. This bit has specific power-dependent behavior:
//!
//! - **When set to logic 0**: The oscillator is enabled (running)
//! - **When set to logic 1**: The oscillator is disabled, but **only when the DS3231
//!   switches to battery backup power (VBAT)**
//!
//! ### Important
//!
//! - **Main Power (VCC)**: When powered by VCC, the oscillator is **always running**
//!   regardless of the EOSC bit status
//! - **Battery Power (VBAT)**: The EOSC bit only takes effect during battery backup
//!   operation to conserve power
//! - **Default State**: The EOSC bit is cleared (logic 0) when power is first applied
//!
//! This means that `halt_clock()` will only stop timekeeping when running on battery
//! power, making it primarily useful for extending battery life rather than general
//! clock control.

pub use rtc_hal::control::RtcPowerControl;

use crate::{
    Ds3231,
    registers::{EOSC_BIT, Register},
};

impl<I2C> RtcPowerControl for Ds3231<I2C>
where
    I2C: embedded_hal::i2c::I2c,
{
    /// Start or resume the RTC oscillator so that timekeeping can continue.
    ///
    /// This clears the EOSC bit (sets to logic 0) to enable the oscillator.
    /// The operation is idempotent - calling it when already running has no effect.
    ///
    /// **Note**: When powered by VCC, the oscillator runs regardless of this setting.
    fn start_clock(&mut self) -> Result<(), Self::Error> {
        self.clear_register_bits(Register::Control, EOSC_BIT)
    }

    /// Halt the RTC oscillator to conserve power during battery backup operation.
    ///
    /// This sets the EOSC bit (sets to logic 1) to disable the oscillator.
    ///
    /// **Important**: This only takes effect when the DS3231 switches to battery
    /// backup power (VBAT). When powered by VCC, the oscillator continues running
    /// regardless of this setting.
    fn halt_clock(&mut self) -> Result<(), Self::Error> {
        self.set_register_bits(Register::Control, EOSC_BIT)
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::registers::{EOSC_BIT, Register};
    use embedded_hal_mock::eh1::i2c::{Mock as I2cMock, Transaction as I2cTransaction};
    use rtc_hal::control::RtcPowerControl;

    const DS3231_ADDR: u8 = 0x68;

    #[test]
    fn test_start_clock_sets_eosc_bit_to_zero() {
        let expectations = vec![
            // Read current control register value with EOSC bit set (oscillator disabled)
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![EOSC_BIT], // EOSC bit is set (oscillator disabled)
            ),
            // Write back with EOSC bit cleared (oscillator enabled)
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), 0b0000_0000]),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.start_clock();
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_start_clock_already_running() {
        let expectations = vec![
            // Read current control register value with EOSC bit already cleared
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b0000_0000], // EOSC bit already cleared
            ),
            // No write transaction needed since bit is already in correct state
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.start_clock();
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_start_clock_preserves_other_bits() {
        let expectations = vec![
            // Read control register with other bits set and EOSC bit set
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b0100_0101 | EOSC_BIT], // Other bits set + EOSC bit
            ),
            // Write back preserving other bits but clearing EOSC bit
            I2cTransaction::write(
                DS3231_ADDR,
                vec![Register::Control.addr(), 0b0100_0101], // Other bits preserved, EOSC cleared
            ),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.start_clock();
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_halt_clock_sets_eosc_bit_to_one() {
        let expectations = vec![
            // Read current control register value with EOSC bit cleared (oscillator enabled)
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b0000_0000], // EOSC bit is cleared
            ),
            // Write back with EOSC bit set (oscillator disabled)
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), EOSC_BIT]),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.halt_clock();
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_halt_clock_already_halted() {
        let expectations = vec![
            // Read current control register value with EOSC bit already set
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![EOSC_BIT], // EOSC bit already set
            ),
            // No write transaction needed since bit is already in correct state
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.halt_clock();
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_halt_clock_preserves_other_bits() {
        let expectations = vec![
            // Read control register with other bits set and EOSC bit cleared
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b0010_1010], // Other bits set, EOSC bit cleared
            ),
            // Write back preserving other bits but setting EOSC bit
            I2cTransaction::write(
                DS3231_ADDR,
                vec![Register::Control.addr(), 0b1010_1010 | EOSC_BIT], // Other bits preserved, EOSC set
            ),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.halt_clock();
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_start_clock_i2c_read_error() {
        let expectations = vec![
            I2cTransaction::write_read(DS3231_ADDR, vec![Register::Control.addr()], vec![0x00])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.start_clock();
        assert!(result.is_err());

        i2c_mock.done();
    }

    #[test]
    fn test_start_clock_i2c_write_error() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![EOSC_BIT], // EOSC bit set, needs clearing
            ),
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), 0b0000_0000])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.start_clock();
        assert!(result.is_err());

        i2c_mock.done();
    }

    #[test]
    fn test_halt_clock_i2c_read_error() {
        let expectations = vec![
            I2cTransaction::write_read(DS3231_ADDR, vec![Register::Control.addr()], vec![0x00])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.halt_clock();
        assert!(result.is_err());

        i2c_mock.done();
    }

    #[test]
    fn test_halt_clock_i2c_write_error() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b0000_0000], // EOSC bit cleared, needs setting
            ),
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), EOSC_BIT])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.halt_clock();
        assert!(result.is_err());

        i2c_mock.done();
    }

    #[test]
    fn test_power_control_sequence_start_halt_start() {
        let expectations = vec![
            // First start_clock() call
            I2cTransaction::write_read(DS3231_ADDR, vec![Register::Control.addr()], vec![EOSC_BIT]),
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), 0b0000_0000]),
            // halt_clock() call
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b0000_0000],
            ),
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), EOSC_BIT]),
            // Second start_clock() call
            I2cTransaction::write_read(DS3231_ADDR, vec![Register::Control.addr()], vec![EOSC_BIT]),
            I2cTransaction::write(DS3231_ADDR, vec![Register::Control.addr(), 0b0000_0000]),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        // Test sequence of operations
        assert!(ds3231.start_clock().is_ok());
        assert!(ds3231.halt_clock().is_ok());
        assert!(ds3231.start_clock().is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_start_clock_clears_only_eosc_bit() {
        // Test that EOSC_BIT has the correct value
        let expectations = vec![
            I2cTransaction::write_read(
                DS3231_ADDR,
                vec![Register::Control.addr()],
                vec![0b1111_1111], // All bits set
            ),
            I2cTransaction::write(
                DS3231_ADDR,
                vec![Register::Control.addr(), !EOSC_BIT], // All bits except EOSC
            ),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds3231 = Ds3231::new(&mut i2c_mock);

        let result = ds3231.start_clock();
        assert!(result.is_ok());

        i2c_mock.done();
    }
}
```
