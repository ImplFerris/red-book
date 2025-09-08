# Core Traits

With our error handling foundation in place, we can now implement the core "Rtc" trait that defines the essential RTC operations:

```rust
pub trait Rtc: ErrorType {
    fn get_datetime(&mut self) -> Result<DateTime, Self::Error>;
    fn set_datetime(&mut self, datetime: &DateTime) -> Result<(), Self::Error>;
}
```

We will define two fundamental functions for RTC interaction:

- get_datetime() - Reads the current date and time from the RTC hardware and converts it into our standardized DateTime struct. The driver handles the hardware-specific details of reading registers and converting between different data formats (like BCD to binary).

- set_datetime() - Accepts a DateTime instance and writes it to the RTC hardware. The driver is responsible for converting the DateTime into the appropriate hardware format and performing the necessary register writes.

> We will shortly define The DateTime struct in the datetime module

Both functions return Result types using Self::Error as the error type. Since `Rtc` extends the supertrait `ErrorType`, any driver implementing Rtc must also implement ErrorType and define their associated error type. This creates a unified error handling system where all RTC operations use consistent error categorization through our Error trait.

## Blanket Implementation for Rtc Trait

We provide a blanket implementation for mutable references to any type that implements Rtc:

```rust
impl<T: Rtc + ?Sized> Rtc for &mut T {
    #[inline]
    fn get_datetime(&mut self) -> Result<DateTime, Self::Error> {
        T::get_datetime(self)
    }

    #[inline]
    fn set_datetime(&mut self, datetime: &DateTime) -> Result<(), Self::Error> {
        T::set_datetime(self, datetime)
    }
}
```

This implementation allows users to pass either owned RTC instances or mutable references to functions that accept Rtc implementations. The ?Sized bound enables this to work with trait objects and other dynamically sized types.

## How Application and Library Developers Use This

Application and library developers who use RTC drivers should take the Rtc instance as an argument to new() and store it in their struct. They should not take &mut Rtc - the trait has a blanket implementation for all &mut T, so taking just Rtc ensures users can still pass a &mut but are not forced to:

```rust
// Correct pattern
pub fn new(rtc: RTC) -> Self {
    Self { rtc }
}

// Avoid this - Forces users to pass mutable references
pub fn new(rtc: &mut Rtc) -> Self { 
    Self { rtc }
}
```

This approach provides better flexibility. Users can pass either owned RTC driver instances or mutable references, giving them the choice of ownership model that works best for their use case.

## The final code for the rtc module

```rust
//! # RTC Trait Interface
//!
//! This module defines the core trait for Real-Time Clock (RTC) devices in embedded systems.
//!
//! ## Features
//! - Provides a platform-independent interface for reading and writing date/time values to hardware RTC chips.
//! - Compatible with the design patterns of `embedded-hal`, focusing on trait-based abstraction.
//! - Uses the hardware-agnostic `DateTime` struct for representing calendar date and time.
//!
//! ## Usage Notes
//! - Each RTC driver should implement its own error type conforming to the `Error` trait, allowing accurate hardware-specific error reporting.
//! - Drivers are responsible for validating that all `DateTime` values provided are within the supported range of their underlying hardware (for example, some chips only support years 2000-2099).
//! - This trait is intended for use in platform implementors and applications needing unified RTC access across hardware targets.
//!
//! ## For application and library developers
//!
//! Applications and libraries should take the `Rtc` instance as an argument to `new()`, and store it in their
//! struct. They **should not** take `&mut Rtc`, the trait has a blanket impl for all `&mut T`
//! so taking just `Rtc` ensures the user can still pass a `&mut`, but is not forced to.
//!
//! Applications and libraries **should not** try to enable sharing by taking `&mut Rtc` at every method.
//! This is much less ergonomic than owning the `Rtc`, which still allows the user to pass an
//! implementation that does sharing behind the scenes.
//!
//! ## Example
//! ```ignore
//! use crate::{datetime::DateTime, error::ErrorType, rtc::Rtc};
//!
//! let mut rtc = Ds1307::new(i2c);
//! let now = rtc.get_datetime()?;
//! rtc.set_datetime(&DateTime::new(2024, 8, 16, 12, 0, 0)?)?;
//! ```
use crate::{datetime::DateTime, error::ErrorType};

/// Core trait for Real-Time Clock (RTC) devices.
///
/// This trait provides a platform-agnostic interface for reading and
/// writing date/time values from hardware RTC chips. It is designed
/// to be similar in style to `embedded-hal` traits.
///
/// Each RTC implementation should define:
/// - An associated error type for hardware-specific errors
///
/// The `DateTime` struct used here is hardware-agnostic. Drivers must
/// validate that provided values fall within the supported range.
///
/// # Example
///
/// ```ignore
/// let mut rtc = Ds1307::new(i2c);
/// let now = rtc.get_datetime()?;
/// rtc.set_datetime(&DateTime::new(2024, 8, 16, 12, 0, 0)?)?;
pub trait Rtc: ErrorType {
    /// Get the current date and time atomically.
    ///
    /// # Errors
    ///
    /// Returns `Self::Error` if communication with the RTC fails.
    fn get_datetime(&mut self) -> Result<DateTime, Self::Error>;

    /// Set the current date and time atomically.
    ///
    /// # Errors
    ///
    /// Returns `Self::Error` if communication with the RTC fails or
    /// if the provided `DateTime` is out of range for this device.
    fn set_datetime(&mut self, datetime: &DateTime) -> Result<(), Self::Error>;
}

/// blanket impl for all `&mut T`
impl<T: Rtc + ?Sized> Rtc for &mut T {
    #[inline]
    fn get_datetime(&mut self) -> Result<DateTime, Self::Error> {
        T::get_datetime(self)
    }

    #[inline]
    fn set_datetime(&mut self, datetime: &DateTime) -> Result<(), Self::Error> {
        T::set_datetime(self, datetime)
    }
}
```
