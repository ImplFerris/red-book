# Registers Module

We will create an enum to represent the registers of the DS3231 RTC:

```rust
#[repr(u8)]
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Register {
    /// Seconds register (0x00) - BCD format 00-59, bit 7 = Clock Halt
    Seconds = 0x00,
    /// Minutes register (0x01) - BCD format 00-59
    Minutes = 0x01,
    /// Hours register (0x02) - BCD format, supports 12/24 hour mode
    Hours = 0x02,
    /// Day of week register (0x03) - 1-7 (Sunday=1)
    Day = 0x03,
    /// Date register (0x04) - BCD format 01-31
    Date = 0x04,
    /// Month register (0x05) - BCD format 01-12
    Month = 0x05,
    /// Year register (0x06) - BCD format 00-99 (2000-2099)
    Year = 0x06,

    /// Control register (0x0E)
    Control = 0x0E,
}

impl Register {
    pub const fn addr(self) -> u8 {
        self as u8
    }
}
```

Next, we will define constants for the register bit flags:

```rust
/// Control register (0x0E) bit flags
/// Enable Oscillator
pub const EOSC_BIT: u8 = 1 << 7;
///  Interrupt Control
pub const INTCN_BIT: u8 = 1 << 2;
/// Rate Select mask
pub const RS_MASK: u8 = 0b0001_1000;
```

EOSC_BIT controls the oscillator, INTCN_BIT controls whether the pin outputs interrupts or square waves, and RS_MASK selects which bits control the square wave frequency. When INTCN_BIT is 0, the pin outputs a square wave; when it's 1, the pin outputs interrupts from alarms.
