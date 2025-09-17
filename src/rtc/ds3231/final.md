# Final

Finally, we have successfully implemented the rtc-hal traits for the DS3231 driver. We will now update lib.rs to re-export the core traits and main struct, providing a clean and convenient public API for users of our driver.

```rust
pub mod control;
pub mod datetime;
mod ds3231;
pub mod error;
pub mod registers;
pub mod square_wave;

// Re-export Ds3231
pub use ds3231::Ds3231;

// Re-export RTC HAL
pub use rtc_hal::{datetime::DateTime, rtc::Rtc};
```

You have now completed the RTC chapter! You built both the DS1307 and DS3231 drivers. Next, i will show you a demo of how to use them.
