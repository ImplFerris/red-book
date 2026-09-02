# Alarm Builder

The `AlarmBuilder` provides methods for creating an `AlarmConfig` and validating it before use.

```rust
pub struct AlarmBuilder {
    config: AlarmConfig,
}
```

## Configuration and Build

Let's add methods to configure the alarm and a `build()` method to validate and create the final `AlarmConfig`.

```rust
impl AlarmBuilder {
    /// Creates an alarm matching the specified hour and minute.
    ///
    /// The hour is specified in 24-hour format (`0..=23`).
    ///
    /// # Example
    ///
    /// ```
    /// # use rtc_hal::alarm::AlarmBuilder;
    /// let alarm = AlarmBuilder::at(7, 30).build()?;
    /// ```
    pub const fn at(hour: u8, minute: u8) -> Self {
        let config = AlarmConfig {
            hour: Some(hour),
            minute: Some(minute),
            second: None,
            weekday: None,
            day_of_month: None,
            month: None,
        };
        Self { config }
    }

    /// Creates an alarm matching the specified hour, minute, and second.
    ///
    /// The hour is specified in 24-hour format (`0..=23`).
    ///
    /// # Example
    ///
    /// ```
    /// # use rtc_hal::alarm::AlarmBuilder;
    /// let alarm = AlarmBuilder::at_hms(7, 30, 15).build()?;
    /// ```
    pub const fn at_hms(hour: u8, minute: u8, second: u8) -> Self {
        let config = AlarmConfig {
            hour: Some(hour),
            minute: Some(minute),
            second: Some(second),
            weekday: None,
            day_of_month: None,
            month: None,
        };
        Self { config }
    }

    /// Creates an alarm matching only the specified hour.
    ///
    /// The hour is specified in 24-hour format (`0..=23`).
    pub const fn at_hour(hour: u8) -> Self {
        let config = AlarmConfig {
            hour: Some(hour),
            minute: None,
            second: None,
            weekday: None,
            day_of_month: None,
            month: None,
        };
        Self { config }
    }

    /// Creates an alarm matching only the specified minute.
    pub const fn at_minute(minute: u8) -> Self {
        let config = AlarmConfig {
            hour: None,
            minute: Some(minute),
            second: None,
            weekday: None,
            day_of_month: None,
            month: None,
        };
        Self { config }
    }

    /// Creates an alarm matching only the specified second.
    pub const fn at_second(second: u8) -> Self {
        let config = AlarmConfig {
            hour: None,
            minute: None,
            second: Some(second),
            weekday: None,
            day_of_month: None,
            month: None,
        };
        Self { config }
    }

    /// Makes the alarm match on the specified weekday.
    pub const fn on(mut self, weekday: Weekday) -> Self {
        self.config.weekday = Some(weekday);
        self
    }

    /// Makes the alarm match on the specified day of the month.
    pub const fn on_day_of_month(mut self, day: u8) -> Self {
        self.config.day_of_month = Some(day);
        self
    }

    /// Makes the alarm match in the specified month.
    ///
    /// The month uses the same `1..=12` representation as
    /// [`crate::datetime::DateTime`].
    pub const fn on_month(mut self, month: u8) -> Self {
        self.config.month = Some(month);
        self
    }

    /// Builds and validates the alarm configuration.
    pub fn build(self) -> Result<AlarmConfig, AlarmConfigError> {
        self.config.validate()?;
        Ok(self.config)
    }
}
```
