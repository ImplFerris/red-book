# Alarm Config

Before defining the trait for the alarm, let's first design the configuration part. We need to create a struct with fields that can cover the alarm combinations supported by most RTCs.

For this, we will create an `AlarmConfig` struct:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[cfg_attr(feature = "defmt", derive(defmt::Format))]
pub struct AlarmConfig {
    second: Option<u8>,
    minute: Option<u8>,
    hour: Option<u8>, // 24 hour format
    weekday: Option<Weekday>,
    day_of_month: Option<u8>,
    month: Option<u8>,
}
```

The fields are `Option` because not every alarm needs to match every field. A configured field is used for matching, while `None` means that field is not used.

## Validation

Each RTC has its own limitations and restrictions for alarms, and these can vary between RTC modules. The RTC HAL cannot validate all of these hardware-specific rules.

Instead, the RTC HAL can validate the generic constraints of the alarm values, such as the valid ranges for seconds, minutes, hours, days, and months. The individual RTC drivers can then validate any additional restrictions required by their hardware.

```rust
/// Errors that can occur when building an alarm configuration.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[cfg_attr(feature = "defmt", derive(defmt::Format))]
pub enum AlarmConfigError {
    /// The seconds value is outside the valid range of `0..=59`.
    InvalidSecond,

    /// The minutes value is outside the valid range of `0..=59`.
    InvalidMinute,

    /// The hour value is outside the valid range of `0..=23`.
    InvalidHour,

    /// The day of the month is outside the valid range of `1..=31`.
    InvalidDayOfMonth,

    /// The month value is outside the valid range of `1..=12`.
    InvalidMonth,
}
```

```rust
impl AlarmConfig {
    fn validate(&self) -> Result<(), AlarmConfigError> {
        if let Some(second) = self.second
            && second > 59
        {
            return Err(AlarmConfigError::InvalidSecond);
        }

        if let Some(minute) = self.minute
            && minute > 59
        {
            return Err(AlarmConfigError::InvalidMinute);
        }

        if let Some(hour) = self.hour
            && hour > 23
        {
            return Err(AlarmConfigError::InvalidHour);
        }

        if let Some(day) = self.day_of_month
            && !(1..=31).contains(&day)
        {
            return Err(AlarmConfigError::InvalidDayOfMonth);
        }

        if let Some(month) = self.month
            && !(1..=12).contains(&month)
        {
            return Err(AlarmConfigError::InvalidMonth);
        }

        Ok(())
    }

    /// Returns the configured second, if any.
    pub const fn second(&self) -> Option<u8> {
        self.second
    }

    /// Returns the configured minute, if any.
    pub const fn minute(&self) -> Option<u8> {
        self.minute
    }

    /// Returns the configured hour, if any.
    pub const fn hour(&self) -> Option<u8> {
        self.hour
    }

    /// Returns the configured weekday, if any.
    pub const fn weekday(&self) -> Option<Weekday> {
        self.weekday
    }

    /// Returns the configured day of the month, if any.
    pub const fn day_of_month(&self) -> Option<u8> {
        self.day_of_month
    }

    /// Returns the configured month, if any.
    pub const fn month(&self) -> Option<u8> {
        self.month
    }
}
```

The user cannot directly create or modify an `AlarmConfig`. This is intentional because we want to prevent an invalid configuration from being created without validation.

Initially, I designed `AlarmConfig` with setter methods that allowed users to modify its values. The fields were still private, but users could change the configuration without calling the validation function afterward. This meant that an invalid configuration could still be created.

To ensure that the alarm configuration is always validated, I introduced the builder pattern. We will see how this works in the next chapter.
