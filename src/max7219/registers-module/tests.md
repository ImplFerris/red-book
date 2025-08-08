
# Tests for Register Module

These are simple tests to verify the basic functionality of the Register and DecodeMode enums. Place them at the bottom of the registers.rs module.

```rust

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_register_addr() {
        assert_eq!(Register::NoOp.addr(), 0x00);
        assert_eq!(Register::Digit0.addr(), 0x01);
        assert_eq!(Register::Digit7.addr(), 0x08);
        assert_eq!(Register::DecodeMode.addr(), 0x09);
        assert_eq!(Register::Intensity.addr(), 0x0A);
        assert_eq!(Register::ScanLimit.addr(), 0x0B);
        assert_eq!(Register::Shutdown.addr(), 0x0C);
        assert_eq!(Register::DisplayTest.addr(), 0x0F);
    }

    #[test]
    fn test_try_digit_valid() {
        assert_eq!(Register::try_digit(0), Ok(Register::Digit0));
        assert_eq!(Register::try_digit(1), Ok(Register::Digit1));
        assert_eq!(Register::try_digit(2), Ok(Register::Digit2));
        assert_eq!(Register::try_digit(3), Ok(Register::Digit3));
        assert_eq!(Register::try_digit(4), Ok(Register::Digit4));
        assert_eq!(Register::try_digit(5), Ok(Register::Digit5));
        assert_eq!(Register::try_digit(6), Ok(Register::Digit6));
        assert_eq!(Register::try_digit(7), Ok(Register::Digit7));
    }

    #[test]
    fn test_try_digit_invalid() {
        assert_eq!(Register::try_digit(8), Err(Error::InvalidDigit));
        assert_eq!(Register::try_digit(255), Err(Error::InvalidDigit));
    }

    #[test]
    fn test_digits_iterator() {
        let expected = [
            Register::Digit0,
            Register::Digit1,
            Register::Digit2,
            Register::Digit3,
            Register::Digit4,
            Register::Digit5,
            Register::Digit6,
            Register::Digit7,
        ];
        let actual: Vec<Register> = Register::digits().collect();
        assert_eq!(actual, expected);
    }

    #[test]
    fn test_decode_mode_value() {
        assert_eq!(DecodeMode::NoDecode.value(), 0x00);
        assert_eq!(DecodeMode::Digit0.value(), 0x01);
        assert_eq!(DecodeMode::Digits0To3.value(), 0x0F);
        assert_eq!(DecodeMode::AllDigits.value(), 0xFF);
    }
}
```
