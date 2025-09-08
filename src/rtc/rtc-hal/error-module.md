# Error Module

Now it is time to put the design into action. In this section, we will implement the error module (error.rs).

## Error Kind

We will define the ErrorKind enum to specify the types of errors that can happen when working with RTC drivers. It covers the common problems you'll run into when using RTC hardware.

```rust
/// Common categories of errors for RTC drivers
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[cfg_attr(feature = "defmt", derive(defmt::Format))]
#[non_exhaustive]
pub enum ErrorKind {
    // Errors related to core traits
    /// Underlying bus error (I2C, SPI, etc.)
    Bus,
    /// Invalid date/time value provided
    InvalidDateTime,

    // Errors related to extended
    /// Invalid alarm configuration
    InvalidAlarmConfig,
    /// The specified square wave frequency is not supported by the RTC
    UnsupportedSqwFrequency,
    /// Invalid register address
    InvalidAddress,
    /// NVRAM address out of bounds
    NvramOutOfBounds,
    /// NVRAM is write protected
    NvramWriteProtected,

    /// Any other error not covered above
    Other,
}
```

We provide optional defmt support for ErrorKind to enable efficient logging in embedded environments. Users can enable this functionality through the "defmt" feature flag.

### `non_exhaustive` attribute

Here, the `#[non_exhaustive]` attribute tells Rust that this enum might get new variants in the future. When the user match on this enum in the downstream crate (i.e the one depends on the rtc-hal crate), user must include a wildcard pattern (like _ => ...). This way, even if we add new error types later, it won't be breaking change. You can find my detailed explanation of this attribute in my blog post [here](https://blog.implrust.com/posts/2025/09/non-exhaustive-attribute-in-rust/).

## Error Trait

Next, we will define the Error trait that provides a standard interface for all RTC driver errors:

```rust
pub trait Error: core::fmt::Debug {
    /// Map a driver-specific error into a general category
    fn kind(&self) -> ErrorKind;
}
```

This trait serves as a bridge between specific driver implementations and the general error categories. Any RTC driver error must implement this trait, which requires:

- Debug trait: All errors must implement Debug so they can be formatted for debugging purposes.

- kind() method: This method maps any driver-specific error into one of the standard ErrorKind categories. This lets higher-level code handle errors in a consistent way, even when different drivers have different internal error types.

## ErrorType Trait

The ErrorType trait defines what error type an RTC driver uses:

```rust
pub trait ErrorType {
    type Error: Error;
}
```

This will be the super trait that the Rtc trait will depend on. This trait is simple but important. It lets RTC drivers specify their error type as an associated type. Having both share the name "Error" might be slightly confusing. Basically, it's telling us that the associated type "Error" must implement the "Error" trait we defined earlier.

### Blanket Implementation

We implement a blanket implementation for mutable references: 

```rust
impl<T: ErrorType + ?Sized> ErrorType for &mut T {
    type Error = T::Error;
}
```

This allows the driver to not need different code for taking ownership vs reference (or they don't have to pick one), they can write general code. Users can pass either ownership or a reference. For a deeper explanation, check out this blog post [here](https://blog.implrust.com/posts/2025/09/blanket-implementation-in-rust/).


### Why Use ErrorType?

This pattern separates error type definition from actual functionality. Currently this separation won't be much helpful, because only the Rtc trait depends on it and all other traits depend on Rtc. However, if we plan to add an async version (or some other variant) of Rtc, we won't need to repeat the ErrorType part. This is how embedded-hal has also created the ErrorType trait, so that it can be used in the embedded-hal-async version as well.


### Using ErrorKind Directly

If the driver has only the errors we defined in ErrorKind (no custom error kinds), we will give flexibility of using the ErrorKind enum itself directly as the Error. To achieve that, we will need to implement the Error trait for ErrorKind as well.

```rust
impl Error for ErrorKind {
    #[inline]
    fn kind(&self) -> ErrorKind {
        *self
    }
}
```

This implementation allows simple drivers to declare:

```rust
type Error = ErrorKind;
```

The kind() method returns the enum variant itself, since ErrorKind already represents the error category. The #[inline] attribute tells the compiler to optimize this away since it's just returning the same value.

We implement the Display trait also for the ErrorKind:

```rust
impl core::fmt::Display for ErrorKind {
    fn fmt(&self, f: &mut core::fmt::Formatter<'_>) -> core::fmt::Result {
        match self {
            Self::Bus => write!(f, "Underlying bus error occurred"),
            Self::InvalidDateTime => write!(f, "Invalid datetime value provided"),
            Self::InvalidAlarmConfig => write!(f, "Invalid alarm configuration"),
            Self::UnsupportedSqwFrequency => write!(
                f,
                "The specified square wave frequency is not supported by the RTC"
            ),
            Self::InvalidAddress => write!(f, "Invalid register address"),
            Self::NvramOutOfBounds => write!(f, "NVRAM address out of bounds"),
            Self::NvramWriteProtected => write!(f, "NVRAM is write protected"),
            Self::Other => write!(
                f,
                "A different error occurred. The original error may contain more information"
            ),
        }
    }
}
```

## The final code (with test):

```rust
//! # RTC Error Types and Classification
//!
//! This module provides a standardized error handling framework for RTC drivers,
//! allowing consistent error categorization across different RTC hardware implementations.

/// Common categories of errors for RTC drivers
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[cfg_attr(feature = "defmt", derive(defmt::Format))]
#[non_exhaustive]
pub enum ErrorKind {
    // Errors related to core traits
    /// Underlying bus error (I2C, SPI, etc.)
    Bus,
    /// Invalid date/time value provided
    InvalidDateTime,

    // Errors related to extended
    /// Invalid alarm configuration
    InvalidAlarmConfig,
    /// The specified square wave frequency is not supported by the RTC
    UnsupportedSqwFrequency,
    /// Invalid register address
    InvalidAddress,
    /// NVRAM address out of bounds
    NvramOutOfBounds,
    /// NVRAM is write protected
    NvramWriteProtected,

    /// Any other error not covered above
    Other,
}

/// Trait that RTC driver error types should implement.
///
/// Allows converting driver-specific errors into standard categories.
/// Drivers can either define custom error types or use `ErrorKind` directly.
pub trait Error: core::fmt::Debug {
    /// Map a driver-specific error into a general category
    fn kind(&self) -> ErrorKind;
}

/// RTC error type trait.
///
/// This just defines the error type, to be used by the other traits.
pub trait ErrorType {
    /// Error type
    type Error: Error;
}

/// Allows `ErrorKind` to be used directly as an error type.
///
/// Simple drivers can use `type Error = ErrorKind` instead of defining custom errors.
impl Error for ErrorKind {
    #[inline]
    fn kind(&self) -> ErrorKind {
        *self
    }
}

// blanket impl for all `&mut T`
impl<T: ErrorType + ?Sized> ErrorType for &mut T {
    type Error = T::Error;
}

impl core::fmt::Display for ErrorKind {
    fn fmt(&self, f: &mut core::fmt::Formatter<'_>) -> core::fmt::Result {
        match self {
            Self::Bus => write!(f, "Underlying bus error occurred"),
            Self::InvalidDateTime => write!(f, "Invalid datetime value provided"),
            Self::InvalidAlarmConfig => write!(f, "Invalid alarm configuration"),
            Self::UnsupportedSqwFrequency => write!(
                f,
                "The specified square wave frequency is not supported by the RTC"
            ),
            Self::InvalidAddress => write!(f, "Invalid register address"),
            Self::NvramOutOfBounds => write!(f, "NVRAM address out of bounds"),
            Self::NvramWriteProtected => write!(f, "NVRAM is write protected"),
            Self::Other => write!(
                f,
                "A different error occurred. The original error may contain more information"
            ),
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    // Mock error type for testing
    #[derive(Debug, Clone, Copy, PartialEq, Eq)]
    enum MockRtcError {
        I2cError,
        InvalidDateTime,
        InvalidAlarmTime,
        UnsupportedSqwFrequency,
        InvalidRegisterAddress,
        NvramAddressOutOfBounds,
        NvramWriteProtected,
        UnknownError,
    }

    impl Error for MockRtcError {
        fn kind(&self) -> ErrorKind {
            match self {
                MockRtcError::I2cError => ErrorKind::Bus,
                MockRtcError::InvalidDateTime => ErrorKind::InvalidDateTime,
                MockRtcError::InvalidAlarmTime => ErrorKind::InvalidAlarmConfig,
                MockRtcError::UnsupportedSqwFrequency => ErrorKind::UnsupportedSqwFrequency,
                MockRtcError::InvalidRegisterAddress => ErrorKind::InvalidAddress,
                MockRtcError::NvramAddressOutOfBounds => ErrorKind::NvramOutOfBounds,
                MockRtcError::NvramWriteProtected => ErrorKind::NvramWriteProtected,
                _ => ErrorKind::Other,
            }
        }
    }

    #[test]
    fn test_error_kind_mapping() {
        assert_eq!(MockRtcError::I2cError.kind(), ErrorKind::Bus);
        assert_eq!(
            MockRtcError::InvalidDateTime.kind(),
            ErrorKind::InvalidDateTime
        );
        assert_eq!(
            MockRtcError::InvalidAlarmTime.kind(),
            ErrorKind::InvalidAlarmConfig
        );
        assert_eq!(
            MockRtcError::UnsupportedSqwFrequency.kind(),
            ErrorKind::UnsupportedSqwFrequency
        );
        assert_eq!(
            MockRtcError::InvalidRegisterAddress.kind(),
            ErrorKind::InvalidAddress
        );
        assert_eq!(
            MockRtcError::NvramAddressOutOfBounds.kind(),
            ErrorKind::NvramOutOfBounds
        );
        assert_eq!(
            MockRtcError::NvramWriteProtected.kind(),
            ErrorKind::NvramWriteProtected
        );
        assert_eq!(MockRtcError::UnknownError.kind(), ErrorKind::Other);
    }

    #[test]
    fn test_error_kind_equality() {
        assert_eq!(ErrorKind::Bus, ErrorKind::Bus);
        assert_ne!(ErrorKind::Bus, ErrorKind::InvalidDateTime);
        assert_ne!(
            ErrorKind::InvalidAlarmConfig,
            ErrorKind::UnsupportedSqwFrequency
        );
        assert_ne!(ErrorKind::NvramOutOfBounds, ErrorKind::NvramWriteProtected);
    }

    #[test]
    fn test_error_kind_returns_self() {
        let error = ErrorKind::Other;
        assert_eq!(error.kind(), ErrorKind::Other);
    }

    #[test]
    fn test_error_kind_display_messages() {
        assert_eq!(
            format!("{}", ErrorKind::Bus),
            "Underlying bus error occurred"
        );

        assert_eq!(
            format!("{}", ErrorKind::InvalidDateTime),
            "Invalid datetime value provided"
        );

        assert_eq!(
            format!("{}", ErrorKind::InvalidAlarmConfig),
            "Invalid alarm configuration"
        );

        assert_eq!(
            format!("{}", ErrorKind::UnsupportedSqwFrequency),
            "The specified square wave frequency is not supported by the RTC"
        );

        assert_eq!(
            format!("{}", ErrorKind::InvalidAddress),
            "Invalid register address"
        );

        assert_eq!(
            format!("{}", ErrorKind::NvramOutOfBounds),
            "NVRAM address out of bounds"
        );

        assert_eq!(
            format!("{}", ErrorKind::NvramWriteProtected),
            "NVRAM is write protected"
        );

        assert_eq!(
            format!("{}", ErrorKind::Other),
            "A different error occurred. The original error may contain more information"
        );
    }
}
```
