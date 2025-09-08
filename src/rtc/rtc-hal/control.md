# Control Trait

This could have been part of the original `Rtc` trait, but I'm not sure if all RTC hardware supports power control yet. So even though this is basic functionality, I've separated it into its own trait. This trait depends on the Rtc supertrait, which means drivers must always implement the Rtc trait first if they want to use this and other similar traits we'll create later.

The `RtcPowerControl` trait provides basic power control functionality for RTC devices. The `start_clock()` function is to start or resume the RTC oscillator so timekeeping can continue, while `halt_clock()` is to pause the oscillator until it's restarted again.

## content of the control.rs file
```rust
//! Power control functionality for RTC devices.

use crate::rtc::Rtc;

/// This trait extends [`Rtc`] with methods to start and halt the RTC clock.
pub trait RtcPowerControl: Rtc {
    /// Start or resume the RTC oscillator so that timekeeping can continue.
    fn start_clock(&mut self) -> Result<(), Self::Error>;

    /// Halt the RTC oscillator, pausing timekeeping until restarted.
    fn halt_clock(&mut self) -> Result<(), Self::Error>;
}
```
