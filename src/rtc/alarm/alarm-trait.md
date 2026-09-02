# Alarm Trait

The `AlarmConfig` struct is the foundation for describing an alarm. Next, we need to define a trait that describes the operations an RTC should provide for working with alarms.

The trait should allow us to configure an alarm, enable and disable it, check whether it has been triggered, and clear the alarm flag.

Since an RTC can have multiple alarms, the trait uses an `AlarmId` associated type to identify which alarm we want to operate on.

```rust
/// Provides alarm functionality for an RTC.
pub trait Alarm: ErrorType {
    /// Identifies an alarm supported by the RTC.
    type AlarmId;

    /// Configures an alarm without enabling it.
    ///
    /// Returns an error if the configuration is not supported by the RTC.
    fn set_alarm(&mut self, id: Self::AlarmId, alarm: AlarmConfig) -> Result<(), Self::Error>;

    /// Enables a configured alarm.
    fn enable_alarm(&mut self, id: Self::AlarmId) -> Result<(), Self::Error>;

    /// Disables an alarm without removing its configuration.
    fn disable_alarm(&mut self, id: Self::AlarmId) -> Result<(), Self::Error>;

    /// Returns whether the alarm flag is set.
    fn alarm_triggered(&mut self, id: Self::AlarmId) -> Result<bool, Self::Error>;

    /// Clears the alarm flag without disabling the alarm.
    fn clear_alarm(&mut self, id: Self::AlarmId) -> Result<(), Self::Error>;
}
```

As with the `Rtc` trait, we also provide a blanket implementation for `&mut T`.

```rust
/// Blanket implementation for `&mut T`.
impl<T: Alarm + ?Sized> Alarm for &mut T {
    type AlarmId = T::AlarmId;

    #[inline]
    fn set_alarm(&mut self, id: Self::AlarmId, alarm: AlarmConfig) -> Result<(), Self::Error> {
        T::set_alarm(self, id, alarm)
    }

    #[inline]
    fn enable_alarm(&mut self, id: Self::AlarmId) -> Result<(), Self::Error> {
        T::enable_alarm(self, id)
    }

    #[inline]
    fn disable_alarm(&mut self, id: Self::AlarmId) -> Result<(), Self::Error> {
        T::disable_alarm(self, id)
    }

    #[inline]
    fn alarm_triggered(&mut self, id: Self::AlarmId) -> Result<bool, Self::Error> {
        T::alarm_triggered(self, id)
    }

    #[inline]
    fn clear_alarm(&mut self, id: Self::AlarmId) -> Result<(), Self::Error> {
        T::clear_alarm(self, id)
    }
}
```
