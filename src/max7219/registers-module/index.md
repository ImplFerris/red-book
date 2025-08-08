# Registers Module

In this chapter, we will work on the registers module(registers.rs). We could use the register values directly, but using enums is more idiomatic and provides better type safety. We will define two enums here: `Register` and `DecodeMode`. 

The Register enum holds the addresses of each digit register and control register values, which we saw in the [data transfer](../data-transfer/index.md) section. The DecodeMode enum is for configuring automatic digit decoding on seven-segment displays. For LED matrix displays, it should be set to `NoDecode`.

Add the neceessary imports for the module:

```rust
use crate::{Result, error::Error};
```

## The Register Enum

In this enum, we declare all the MAX7219 registers, with each name matching its register address. To make working with these registers easier, we also implement some basic traits that let us copy and compare the register values. 

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(u8)]
pub enum Register {
    /// No-op register
    NoOp = 0x00,
    /// Digit 0 register
    Digit0 = 0x01,
    /// Digit 1 register
    Digit1 = 0x02,
    /// Digit 2 register
    Digit2 = 0x03,
    /// Digit 3 register
    Digit3 = 0x04,
    /// Digit 4 register
    Digit4 = 0x05,
    /// Digit 5 register
    Digit5 = 0x06,
    /// Digit 6 register
    Digit6 = 0x07,
    /// Digit 7 register
    Digit7 = 0x08,
    /// Decode mode register
    DecodeMode = 0x09,
    /// Intensity register
    Intensity = 0x0A,
    /// Scan limit register
    ScanLimit = 0x0B,
    /// Shutdown register
    Shutdown = 0x0C,
    /// Display test register
    DisplayTest = 0x0F,
}
```

Next, we will add some functions that will be helpful later.

First, we will add the addr method that converts a register into its raw u8 value. This is useful when we need to send the register address to the MAX7219 chip.

Next, we will add try_digit function that takes a number and tries to convert it into the matching digit register (Digit0 to Digit7). If the number is out of range, it returns an invalid digit error. If you are wondering why I didn't implement the TryFrom trait, it is because I want to accept only digit registers. Later in the code, when the user sends a digit as input, we need a function to check if it is a valid digit.  By the way, this is an internal function we will use only inside the crate to validate user input. So, we mark it with pub(crate).

We also declare another helper function that gives us an iterator. This returns all the digit registers. This is useful when you want to loop through all digits - for example, to update every row or column on the display.


```rust

impl Register {
    /// Convert register to u8 value
    pub const fn addr(self) -> u8 {
        self as u8
    }

    /// Try to convert a digit index (0-7) into a corresponding `Register::DigitN`.
    pub(crate) fn try_digit(digit: u8) -> Result<Self> {
        match digit {
            0 => Ok(Register::Digit0),
            1 => Ok(Register::Digit1),
            2 => Ok(Register::Digit2),
            3 => Ok(Register::Digit3),
            4 => Ok(Register::Digit4),
            5 => Ok(Register::Digit5),
            6 => Ok(Register::Digit6),
            7 => Ok(Register::Digit7),
            _ => Err(Error::InvalidDigit),
        }
    }

    /// Returns an iterator over all digit registers (Digit0 to Digit7).
    ///
    /// Useful for iterating through display rows or columns when writing
    /// to all digits of a MAX7219 device in order.
    pub fn digits() -> impl Iterator<Item = Register> {
        [
            Register::Digit0,
            Register::Digit1,
            Register::Digit2,
            Register::Digit3,
            Register::Digit4,
            Register::Digit5,
            Register::Digit6,
            Register::Digit7,
        ]
        .into_iter()
    }
}
```
