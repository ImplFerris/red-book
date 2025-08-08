# Test Code for Error Handling

Nothing much happening in this test. We're not even using embedded-hal-mock crate here. Just normal test code. You can paste this at the bottom of the error.rs module.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // Mock SPI error for testing
    #[derive(Debug)]
    struct MockSpiError;

    impl core::fmt::Display for MockSpiError {
        fn fmt(&self, f: &mut core::fmt::Formatter<'_>) -> core::fmt::Result {
            write!(f, "Mock SPI error")
        }
    }

    impl embedded_hal::spi::Error for MockSpiError {
        fn kind(&self) -> embedded_hal::spi::ErrorKind {
            embedded_hal::spi::ErrorKind::Other
        }
    }

    #[test]
    fn test_error_device() {
        assert_eq!(
            format!("{}", Error::InvalidDeviceCount),
            "Invalid device count"
        );
        assert_eq!(
            format!("{}", Error::InvalidScanLimit),
            "Invalid scan limit value"
        );
        assert_eq!(
            format!("{}", Error::InvalidRegister),
            "Invalid register address"
        );
        assert_eq!(
            format!("{}", Error::InvalidDeviceIndex),
            "Invalid device index"
        );
        assert_eq!(format!("{}", Error::InvalidDigit), "Invalid digit");
        assert_eq!(
            format!("{}", Error::InvalidIntensity),
            "Invalid intensity value"
        );
        assert_eq!(format!("{}", Error::SpiError), "SPI communication error");
    }

    #[test]
    fn test_error_debug() {
        // Test that Debug trait is implemented and works
        let error = Error::InvalidDigit;
        let debug_output = format!("{error:?}",);
        assert!(debug_output.contains("InvalidDigit"));
    }

    #[test]
    fn test_from_spi_error() {
        let spi_error = MockSpiError;
        let error = Error::from(spi_error);
        assert_eq!(error, Error::SpiError);
    }

    #[test]
    fn test_error_partialeq() {
        // Test that all variants implement PartialEq correctly
        assert!(Error::InvalidDeviceCount.eq(&Error::InvalidDeviceCount));
        assert!(!Error::InvalidDeviceCount.eq(&Error::InvalidScanLimit));
    }
}
```
