# Error Handling Implementation

In this section, we will implement error handling for our DS1307 driver.

We will start by defining a custom error enum that covers all possible error types we may encounter when working with the DS1307 driver.

```rust
use rtc_hal::datetime::DateTimeError;

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[cfg_attr(feature = "defmt", derive(defmt::Format))]
pub enum Error<I2cError>
where
    I2cError: core::fmt::Debug,
{
    I2c(I2cError),
    InvalidAddress,
    UnsupportedSqwFrequency,
    DateTime(DateTimeError),
    NvramOutOfBounds,
}
```

The `I2c(I2cError)` variant wraps errors that come from the underlying I2C communication layer. Since different microcontroller HALs have different I2C error types, we make our error enum generic over I2cError.  When an I2C operation fails (bus error, timeout, etc.), we wrap that specific error in our I2c variant and propagate it up. This preserves the original error information.

Next, we will implement the Display trait for our error enum to provide clear error messages when debugging or handling errors:

```rust

impl<I2cError> core::fmt::Display for Error<I2cError>
where
    I2cError: core::fmt::Debug + core::fmt::Display,
{
    fn fmt(&self, f: &mut core::fmt::Formatter<'_>) -> core::fmt::Result {
        match self {
            Error::I2c(e) => write!(f, "I2C communication error: {e}"),
            Error::InvalidAddress => write!(f, "Invalid NVRAM address"),
            Error::DateTime(e) => write!(f, "Invalid date/time values: {e}"),
            Error::UnsupportedSqwFrequency => write!(f, "Unsupported square wave frequency"),
            Error::NvramOutOfBounds => write!(f, "NVRAM operation out of bounds"),
        }
    }
}
```

## Implementing Rust's core Error trait

We will implement Rust's core Error trait for our custom error type. Since our error enum already implements Debug and Display, we can use an empty implementation block. Don't confuse this with our rtc-hal's Error trait, which we will implement shortly.

> From docs: Implementing the `Error` trait only requires that `Debug` and `Display` are implemented too.

```rust
impl<I2cError> core::error::Error for Error<I2cError> where
    I2cError: core::fmt::Debug + core::fmt::Display
{
}
```

## Implementing RTC HAL trait

We will implement the rtc-hal's Error trait for our custom error type. As you already know, this trait requires us to implement the kind method that maps our error variants to the standard error kinds we defined in the RTC HAL.

```rust
impl<I2cError> rtc_hal::error::Error for Error<I2cError>
where
    I2cError: core::fmt::Debug,
{
    fn kind(&self) -> rtc_hal::error::ErrorKind {
        match self {
            Error::I2c(_) => rtc_hal::error::ErrorKind::Bus,
            Error::InvalidAddress => rtc_hal::error::ErrorKind::InvalidAddress,
            Error::DateTime(_) => rtc_hal::error::ErrorKind::InvalidDateTime,
            Error::NvramOutOfBounds => rtc_hal::error::ErrorKind::NvramOutOfBounds,
            Error::UnsupportedSqwFrequency => rtc_hal::error::ErrorKind::UnsupportedSqwFrequency,
        }
    }
}
```

## Converting I2C Errors

We finally implement the `From` trait to automatically convert I2C errors into our custom error type. This allows us to use the ? operator when working with I2C operations.

```rust
impl<I2cError> From<I2cError> for Error<I2cError>
where
    I2cError: core::fmt::Debug,
{
    fn from(value: I2cError) -> Self {
        Error::I2c(value)
    }
}
```

## The full code for the Error module (error.rs)

```rust
//! Error type definitions for the DS1307 RTC driver.
//!
//! This module defines the `Error` enum and helper functions
//! for classifying and handling DS1307-specific failures.

use rtc_hal::datetime::DateTimeError;

/// DS1307 driver errors
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[cfg_attr(feature = "defmt", derive(defmt::Format))]
pub enum Error<I2cError>
where
    I2cError: core::fmt::Debug,
{
    /// I2C communication error
    I2c(I2cError),
    /// Invalid register address
    InvalidAddress,
    /// The specified square wave frequency is not supported by the RTC
    UnsupportedSqwFrequency,
    /// Invalid date/time parameters provided by user
    DateTime(DateTimeError),
    /// NVRAM write would exceed available space
    NvramOutOfBounds,
}

impl<I2cError> core::fmt::Display for Error<I2cError>
where
    I2cError: core::fmt::Debug + core::fmt::Display,
{
    fn fmt(&self, f: &mut core::fmt::Formatter<'_>) -> core::fmt::Result {
        match self {
            Error::I2c(e) => write!(f, "I2C communication error: {e}"),
            Error::InvalidAddress => write!(f, "Invalid NVRAM address"),
            Error::DateTime(e) => write!(f, "Invalid date/time values: {e}"),
            Error::UnsupportedSqwFrequency => write!(f, "Unsupported square wave frequency"),
            Error::NvramOutOfBounds => write!(f, "NVRAM operation out of bounds"),
        }
    }
}

impl<I2cError> core::error::Error for Error<I2cError> where
    I2cError: core::fmt::Debug + core::fmt::Display
{
}

/// Converts an [`I2cError`] into an [`Error`] by wrapping it in the
/// [`Error::I2c`] variant.
///
impl<I2cError> From<I2cError> for Error<I2cError>
where
    I2cError: core::fmt::Debug,
{
    fn from(value: I2cError) -> Self {
        Error::I2c(value)
    }
}

impl<I2cError> rtc_hal::error::Error for Error<I2cError>
where
    I2cError: core::fmt::Debug,
{
    fn kind(&self) -> rtc_hal::error::ErrorKind {
        match self {
            Error::I2c(_) => rtc_hal::error::ErrorKind::Bus,
            Error::InvalidAddress => rtc_hal::error::ErrorKind::InvalidAddress,
            Error::DateTime(_) => rtc_hal::error::ErrorKind::InvalidDateTime,
            Error::NvramOutOfBounds => rtc_hal::error::ErrorKind::NvramOutOfBounds,
            Error::UnsupportedSqwFrequency => rtc_hal::error::ErrorKind::UnsupportedSqwFrequency,
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use rtc_hal::datetime::DateTimeError;
    use rtc_hal::error::{Error as RtcError, ErrorKind};

    #[test]
    fn test_from_i2c_error() {
        #[derive(Debug, PartialEq, Eq)]
        struct DummyI2cError(u8);

        let e = Error::from(DummyI2cError(42));
        assert_eq!(e, Error::I2c(DummyI2cError(42)));
    }

    #[test]
    fn test_error_kind_mappings() {
        // I2c variant
        let e: Error<&str> = Error::I2c("oops");
        assert_eq!(e.kind(), ErrorKind::Bus);

        // InvalidAddress
        let e: Error<&str> = Error::InvalidAddress;
        assert_eq!(e.kind(), ErrorKind::InvalidAddress);

        // DateTime
        let e: Error<&str> = Error::DateTime(DateTimeError::InvalidDay);
        assert_eq!(e.kind(), ErrorKind::InvalidDateTime);

        // UnsupportedSqwFrequency
        let e: Error<&str> = Error::UnsupportedSqwFrequency;
        assert_eq!(e.kind(), ErrorKind::UnsupportedSqwFrequency);

        // NvramOutOfBounds
        let e: Error<&str> = Error::NvramOutOfBounds;
        assert_eq!(e.kind(), ErrorKind::NvramOutOfBounds);
    }

    #[derive(Debug, PartialEq, Eq)]
    struct MockI2cError {
        code: u8,
        message: &'static str,
    }

    impl core::fmt::Display for MockI2cError {
        fn fmt(&self, f: &mut core::fmt::Formatter<'_>) -> core::fmt::Result {
            write!(f, "I2C Error {}: {}", self.code, self.message)
        }
    }

    #[test]
    fn test_display_all_variants() {
        let errors = vec![
            (
                Error::I2c(MockI2cError {
                    code: 1,
                    message: "test",
                }),
                "I2C communication error: I2C Error 1: test",
            ),
            (Error::InvalidAddress, "Invalid NVRAM address"),
            (
                Error::DateTime(DateTimeError::InvalidMonth),
                "Invalid date/time values: invalid month",
            ),
            (
                Error::UnsupportedSqwFrequency,
                "Unsupported square wave frequency",
            ),
            (Error::NvramOutOfBounds, "NVRAM operation out of bounds"),
        ];

        for (error, expected) in errors {
            assert_eq!(format!("{error}"), expected);
        }
    }
}
```s
