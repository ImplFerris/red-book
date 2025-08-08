
# The DecodeMode Enum

This enum defines how the MAX7219 decodes the values you send to it for each digit.

The MAX7219 can do something called Code B decoding. When enabled, it automatically converts numbers and few letters (like 0-9, E, H, L, and some others) into the correct 7-segment patterns. This means you don't have to manually control each LED segment yourself.

The DecodeMode enum lets you choose which digits use this automatic decoding and which digits you want to control manually with raw segment data.

- NoDecode means all digits are manual; you control every segment yourself. This is the settings normally used with LED Matrix Displays.

- Digit0 enables decoding only for the first digit.

- Digits0To3 enables decoding for digits 0 to 3; often used for 4-digit 7-segment displays.

- AllDigits turns on decoding for all eight digits.

```rust
/// Decode mode configuration for the MAX7219 display driver.
///
/// Code B decoding allows the driver to automatically convert certain values
/// (such as 0-9, E, H, L, and others) into their corresponding 7-segment patterns.
/// Digits not using Code B must be controlled manually using raw segment data.
///
/// Use this to configure which digits should use Code B decoding and which
/// should remain in raw segment mode.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(u8)]
pub enum DecodeMode {
    /// Disable Code B decoding for all digits (DIG0 to DIG7).
    ///
    /// In this mode, you must manually set each segment (A to G and DP)
    /// using raw segment data.
    NoDecode = 0x00,

    /// Enable Code B decoding for only digit 0 (DIG0).
    ///
    /// All other digits (DIG1 to DIG7) must be controlled manually.
    Digit0 = 0x01,

    /// Enable Code B decoding for digits 0 through 3 (DIG0 to DIG3).
    ///
    /// This is commonly used for 4-digit numeric displays.
    Digits0To3 = 0x0F,

    /// Enable Code B decoding for all digits (DIG0 to DIG7).
    ///
    /// This is typically used for full 8-digit numeric displays.
    AllDigits = 0xFF,
}
```

We also add a simple method value() that converts the enum into the number you need to send to the MAX7219 decode mode register.

```rust
impl DecodeMode {
    /// Convert decode mode to u8 value
    pub const fn value(self) -> u8 {
        self as u8
    }
}
```
