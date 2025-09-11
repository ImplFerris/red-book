# Final

Congratulations! You have successfully completed the chapter and implemented the rtc-hal for the DS1307 driver.

We will now update lib.rs to re-export the core traits and main struct, providing a clean and intuitive public API for users of our driver:

```rust

pub mod control;
pub mod datetime;
mod ds1307;
pub mod error;
pub mod nvram;
pub mod registers;
pub mod square_wave;

// Re-export Ds1307
pub use ds1307::Ds1307;

// Re-export RTC HAL
pub use rtc_hal::{datetime::DateTime, rtc::Rtc};
```

We have not finished the RTC chapter yet. We are yet to implement the DS3231 driver. Then finally we will show you the demo of using them. If you are curious, you can now itslef use the DS1307 library you created with your microcontroller HAL and test it.
