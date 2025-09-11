# impl RtcPowerControl

Now that we have implemented the core Rtc trait, we can extend our DS1307 driver with additional feature-specific traits. Let's start with the `RtcPowerControl` trait, which allows us to halt or start the clock oscillator.

The implementation is quite simple. We just set the Clock Halt bit with the help of the "set_register_bits" method to halt the clock. We clear the Clock Halt bit with the help of the "clear_register_bits" method to start the clock.

```rust
impl<I2C, E> RtcPowerControl for Ds1307<I2C>
where
    I2C: embedded_hal::i2c::I2c<Error = E>,
    E: core::fmt::Debug,
{
    fn start_clock(&mut self) -> Result<(), Self::Error> {
        self.clear_register_bits(Register::Seconds, CH_BIT)
    }

    fn halt_clock(&mut self) -> Result<(), Self::Error> {
        self.set_register_bits(Register::Seconds, CH_BIT)
    }
}
```

## The Full code for the control module (control.rs)

```rust
//! Power control implementation for the DS1307
//!
//! This module provides power management functionality for the DS1307 RTC chip,
//! implementing the `RtcPowerControl` trait to allow starting and stopping the
//! internal oscillator that drives timekeeping operations.
//!
//! The DS1307 uses a Clock Halt (CH) bit in the seconds register to control
//! oscillator operation. When set, the oscillator stops and timekeeping is
//! paused. When cleared, the oscillator runs and time advances normally.

pub use rtc_hal::control::RtcPowerControl;

use crate::{
    Ds1307,
    registers::{CH_BIT, Register},
};

impl<I2C, E> RtcPowerControl for Ds1307<I2C>
where
    I2C: embedded_hal::i2c::I2c<Error = E>,
    E: core::fmt::Debug,
{
    /// Start or resume the RTC oscillator so that timekeeping can continue.
    /// This operation is idempotent - calling it when already running has no effect.
    fn start_clock(&mut self) -> Result<(), Self::Error> {
        // Clear Clock Halt (CH) bit in seconds register to start oscillator
        self.clear_register_bits(Register::Seconds, CH_BIT)
    }

    /// Halt the RTC oscillator, pausing timekeeping until restarted.
    /// This operation is idempotent - calling it when already halted has no effect.
    fn halt_clock(&mut self) -> Result<(), Self::Error> {
        // Set Clock Halt (CH) bit in seconds register to stop oscillator
        self.set_register_bits(Register::Seconds, CH_BIT)
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::registers::{CH_BIT, Register};
    use embedded_hal_mock::eh1::i2c::{Mock as I2cMock, Transaction as I2cTransaction};
    use rtc_hal::control::RtcPowerControl;

    const DS1307_ADDR: u8 = 0x68;

    #[test]
    fn test_start_clock_clears_ch_bit() {
        let expectations = vec![
            // Read current seconds register value with CH bit set (oscillator halted)
            I2cTransaction::write_read(
                DS1307_ADDR,
                vec![Register::Seconds.addr()],
                vec![CH_BIT], // CH bit is set (oscillator halted)
            ),
            // Write back with CH bit cleared (oscillator enabled)
            I2cTransaction::write(DS1307_ADDR, vec![Register::Seconds.addr(), 0b0000_0000]),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(&mut i2c_mock);

        let result = ds1307.start_clock();
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_start_clock_already_running() {
        let expectations = vec![
            // Read current seconds register value with CH bit already cleared
            I2cTransaction::write_read(
                DS1307_ADDR,
                vec![Register::Seconds.addr()],
                vec![0b0000_0000], // CH bit already cleared
            ),
            // No write transaction needed since bit is already in correct state
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(&mut i2c_mock);

        let result = ds1307.start_clock();
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_start_clock_preserves_seconds_value() {
        let expectations = vec![
            // Read seconds register with time value and CH bit set
            I2cTransaction::write_read(
                DS1307_ADDR,
                vec![Register::Seconds.addr()],
                vec![0b0010_1010 | CH_BIT],
            ),
            // Write back preserving seconds value but clearing CH bit
            I2cTransaction::write(
                DS1307_ADDR,
                vec![Register::Seconds.addr(), 0b0010_1010], // Seconds preserved, CH cleared
            ),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(&mut i2c_mock);

        let result = ds1307.start_clock();
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_halt_clock_sets_ch_bit() {
        let expectations = vec![
            // Read current seconds register value with CH bit cleared (oscillator running)
            I2cTransaction::write_read(
                DS1307_ADDR,
                vec![Register::Seconds.addr()],
                vec![0b0000_0000], // CH bit is cleared
            ),
            // Write back with CH bit set (oscillator halted)
            I2cTransaction::write(DS1307_ADDR, vec![Register::Seconds.addr(), CH_BIT]),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(&mut i2c_mock);

        let result = ds1307.halt_clock();
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_halt_clock_already_halted() {
        let expectations = vec![
            // Read current seconds register value with CH bit already set
            I2cTransaction::write_read(
                DS1307_ADDR,
                vec![Register::Seconds.addr()],
                vec![CH_BIT], // CH bit already set
            ),
            // No write transaction needed since bit is already in correct state
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(&mut i2c_mock);

        let result = ds1307.halt_clock();
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_halt_clock_preserves_seconds_value() {
        let expectations = vec![
            // Read seconds register with time value and CH bit cleared
            I2cTransaction::write_read(
                DS1307_ADDR,
                vec![Register::Seconds.addr()],
                vec![0b0101_1001], // CH bit cleared
            ),
            // Write back preserving seconds value but setting CH bit
            I2cTransaction::write(
                DS1307_ADDR,
                vec![Register::Seconds.addr(), 0b0101_1001 | CH_BIT], // Seconds preserved, CH set
            ),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(&mut i2c_mock);

        let result = ds1307.halt_clock();
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_start_clock_i2c_read_error() {
        let expectations = vec![
            I2cTransaction::write_read(DS1307_ADDR, vec![Register::Seconds.addr()], vec![0x00])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(&mut i2c_mock);

        let result = ds1307.start_clock();
        assert!(result.is_err());

        i2c_mock.done();
    }

    #[test]
    fn test_start_clock_i2c_write_error() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS1307_ADDR,
                vec![Register::Seconds.addr()],
                vec![CH_BIT], // CH bit set, needs clearing
            ),
            I2cTransaction::write(DS1307_ADDR, vec![Register::Seconds.addr(), 0b0000_0000])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(&mut i2c_mock);

        let result = ds1307.start_clock();
        assert!(result.is_err());

        i2c_mock.done();
    }

    #[test]
    fn test_halt_clock_i2c_read_error() {
        let expectations = vec![
            I2cTransaction::write_read(DS1307_ADDR, vec![Register::Seconds.addr()], vec![0x00])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(&mut i2c_mock);

        let result = ds1307.halt_clock();
        assert!(result.is_err());

        i2c_mock.done();
    }

    #[test]
    fn test_halt_clock_i2c_write_error() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS1307_ADDR,
                vec![Register::Seconds.addr()],
                vec![0b0000_0000], // CH bit cleared, needs setting
            ),
            I2cTransaction::write(DS1307_ADDR, vec![Register::Seconds.addr(), CH_BIT])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(&mut i2c_mock);

        let result = ds1307.halt_clock();
        assert!(result.is_err());

        i2c_mock.done();
    }

    #[test]
    fn test_power_control_sequence_start_halt_start() {
        let expectations = vec![
            // First start_clock() call
            I2cTransaction::write_read(DS1307_ADDR, vec![Register::Seconds.addr()], vec![CH_BIT]),
            I2cTransaction::write(DS1307_ADDR, vec![Register::Seconds.addr(), 0b0000_0000]),
            // halt_clock() call
            I2cTransaction::write_read(
                DS1307_ADDR,
                vec![Register::Seconds.addr()],
                vec![0b0000_0000],
            ),
            I2cTransaction::write(DS1307_ADDR, vec![Register::Seconds.addr(), CH_BIT]),
            // Second start_clock() call
            I2cTransaction::write_read(DS1307_ADDR, vec![Register::Seconds.addr()], vec![CH_BIT]),
            I2cTransaction::write(DS1307_ADDR, vec![Register::Seconds.addr(), 0b0000_0000]),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(&mut i2c_mock);

        // Test sequence of operations
        assert!(ds1307.start_clock().is_ok());
        assert!(ds1307.halt_clock().is_ok());
        assert!(ds1307.start_clock().is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_start_clock_clears_only_ch_bit() {
        // Test that CH_BIT has the correct value and only it gets cleared
        let expectations = vec![
            I2cTransaction::write_read(
                DS1307_ADDR,
                vec![Register::Seconds.addr()],
                vec![0b1111_1111], // All bits set including CH bit
            ),
            I2cTransaction::write(
                DS1307_ADDR,
                vec![Register::Seconds.addr(), !CH_BIT], // All bits except CH
            ),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(&mut i2c_mock);

        let result = ds1307.start_clock();
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_halt_clock_sets_only_ch_bit() {
        // Test that CH_BIT has the correct value and only it gets set
        let expectations = vec![
            I2cTransaction::write_read(
                DS1307_ADDR,
                vec![Register::Seconds.addr()],
                vec![0b0000_0000], // No bits set
            ),
            I2cTransaction::write(
                DS1307_ADDR,
                vec![Register::Seconds.addr(), CH_BIT], // Only CH bit set
            ),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(&mut i2c_mock);

        let result = ds1307.halt_clock();
        assert!(result.is_ok());

        i2c_mock.done();
    }

    #[test]
    fn test_power_control_with_valid_bcd_seconds() {
        let expectations = vec![
            I2cTransaction::write_read(
                DS1307_ADDR,
                vec![Register::Seconds.addr()],
                vec![0b0010_0101 | CH_BIT],
            ),
            I2cTransaction::write(
                DS1307_ADDR,
                vec![Register::Seconds.addr(), 0b0010_0101], // 25 seconds preserved, CH cleared
            ),
            // halt_clock() - should preserve the 25 seconds value
            I2cTransaction::write_read(
                DS1307_ADDR,
                vec![Register::Seconds.addr()],
                vec![0b0010_0101], // 25 seconds, CH clear
            ),
            I2cTransaction::write(
                DS1307_ADDR,
                vec![Register::Seconds.addr(), 0b0010_0101 | CH_BIT], // 25 seconds + CH bit
            ),
        ];

        let mut i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(&mut i2c_mock);

        assert!(ds1307.start_clock().is_ok());
        assert!(ds1307.halt_clock().is_ok());

        i2c_mock.done();
    }
}
```
