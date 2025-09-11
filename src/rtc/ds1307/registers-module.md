# Registers Module

We will create an enum to represent the registers of the DS1307 RTC:

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
    /// Control register (0x07) - Square wave and output control
    Control = 0x07,
}
```

We will create addr() method that allows to get the register address as a u8 value for I2C communication. We have marked it as const fn so it can be evaluated at compile time. Since we have marked the enum #[repr(u8)], we can just cast as u8 for the conversion.

```rust
impl Register {
    pub const fn addr(self) -> u8 {
        self as u8
    }
}
```

We will define constants for the register bit flags that control various DS1307 features:

```rust
/// Seconds register (0x00) bit flags
pub const CH_BIT: u8 = 0b1000_0000; // Clock Halt

/// Control register (0x07) bit flags
///  Square Wave Enable
pub const SQWE_BIT: u8 = 0b0001_0000;
/// Output Level
pub const OUT_BIT: u8 = 0b1000_0000;
/// Rate Select mask
pub const RS_MASK: u8 = 0b0000_0011;
```

The Clock Halt bit (CH_BIT) in the seconds register stops the oscillator when set. The control register bits manage the square wave output: SQWE_BIT enables/disables the Square wave output, OUT_BIT sets the output level when square wave is disabled, and RS_MASK selects the square wave frequency.

We will also define constants for the NVRAM memory layout:

```rust
/// DS1307 NVRAM starts at register 0x08
pub const NVRAM_START: u8 = 0x08;
/// DS1307 has 56 bytes of NVRAM (0x08-0x3F)
pub const NVRAM_SIZE: u8 = 56;
/// 56 NVRAM + 1 address byte
pub const MAX_NVRAM_WRITE: usize = 57;
```

## The Full code for the registers module(registers.rs)

```rust
//! DS1307 Registers

/// DS1307 Registers
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
    /// Control register (0x07) - Square wave and output control
    Control = 0x07,
}

impl Register {
    /// Returns the raw 7-bit register address as `u8`.
    pub const fn addr(self) -> u8 {
        self as u8
    }
}

/// Seconds register (0x00) bit flags
pub const CH_BIT: u8 = 0b1000_0000; // Clock Halt

/// Control register (0x07) bit flags
///  Square Wave Enable
pub const SQWE_BIT: u8 = 0b0001_0000;
/// Output Level
pub const OUT_BIT: u8 = 0b1000_0000;
/// Rate Select mask
pub const RS_MASK: u8 = 0b0000_0011;

/// DS1307 NVRAM starts at register 0x08
pub const NVRAM_START: u8 = 0x08;
/// DS1307 has 56 bytes of NVRAM (0x08-0x3F)
pub const NVRAM_SIZE: u8 = 56;
/// 56 NVRAM + 1 address byte
pub const MAX_NVRAM_WRITE: usize = 57;
```
