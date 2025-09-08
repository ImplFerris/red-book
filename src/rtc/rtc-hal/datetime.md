# DateTime module (datetime.rs)

The DateTime module provides a simple date and time struct for RTC drivers. It stores year, month, day, hour, minute, and second values with built-in validation to prevent invalid dates. I won't go through this module step by step since it's pretty straightforward - just basic validation and getters/setters with nothing particularly complex happening here. I want you to go through the code once and understand before proceeding.

The main DateTime struct has private fields to force validation through the new() constructor. This ensures you can't create invalid dates like February 30th or hours greater than 23. The validation checks leap years, days per month, and all time component ranges.

The module supports years from 1970 onwards, which works with most RTC chips. Individual drivers can add stricter limits if their hardware has a smaller range (like DS1307 which only supports 2000-2099).

The struct provides getter methods for all fields and setter methods that re-validate when you change values. For example, if you try to change January 31st to February, it will fail validation and keep the original date unchanged.

Additional features include weekday calculation using a standard algorithm, leap year detection, and utility functions for days in each month. The Weekday enum uses Sunday=1 to Saturday=7 numbering, which drivers can convert if their hardware uses different numbering.

```rust
//! # DateTime Module
//!
//! This module defines a `DateTime` struct and helper functions for representing,
//! validating, and working with calendar date and time values in embedded systems.
//!
//! ## Features
//! - Stores year, month, day, hour, minute, second
//! - Built-in validation for all fields (including leap years and month lengths)
//! - Setter and getter methods that enforce validity
//! - Utility functions for leap year detection, days in a month, and weekday calculation
//!
//! ## Year Range
//! The default supported range is **year >= 1970**, which covers the widest set of
//! popular RTC chips. For example:
//!
//! - DS1307, DS3231: 2000-2099
//!
//! Drivers are responsible for checking and enforcing the *exact* year range of the
//! underlying hardware. The `DateTime` type itself only enforces the lower bound (1970)
//! to remain reusable in contexts outside RTCs.
//!
//! ## Weekday Format
//! - This module uses **1=Sunday to 7=Saturday**
//! - Drivers must handle conversion if required

/// Errors that can occur when working with DateTime
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[cfg_attr(feature = "defmt", derive(defmt::Format))]
pub enum DateTimeError {
    /// Invalid month value
    InvalidMonth,
    /// Invalid day value
    InvalidDay,
    /// Invalid hour value
    InvalidHour,
    /// Invalid minute value
    InvalidMinute,
    /// Invalid second value
    InvalidSecond,
    /// Invalid weekday value
    InvalidWeekday,
    /// Invalid Year value
    InvalidYear,
}

impl core::fmt::Display for DateTimeError {
    fn fmt(&self, f: &mut core::fmt::Formatter<'_>) -> core::fmt::Result {
        match self {
            DateTimeError::InvalidMonth => write!(f, "invalid month"),
            DateTimeError::InvalidDay => write!(f, "invalid day"),
            DateTimeError::InvalidHour => write!(f, "invalid hour"),
            DateTimeError::InvalidMinute => write!(f, "invalid minute"),
            DateTimeError::InvalidSecond => write!(f, "invalid second"),
            DateTimeError::InvalidWeekday => write!(f, "invalid weekday"),
            DateTimeError::InvalidYear => write!(f, "invalid year"),
        }
    }
}

impl core::error::Error for DateTimeError {}

/// Date and time representation used across RTC drivers.
///
/// This type represents calendar date and time in a general-purpose way,
/// independent of any specific RTC hardware.
///
/// - Validates that `year >= 1970`
/// - Other limits (e.g., 2000-2099) must be enforced by individual drivers
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct DateTime {
    /// Year (full year, e.g., 2024)
    year: u16,
    /// Month (1-12)
    month: u8,
    /// Day of the month (1-31 depending on month/year)
    day_of_month: u8,
    /// Hour (0-23)
    hour: u8,
    /// Minute (0-59)
    minute: u8,
    /// Second (0-59)
    second: u8,
}

impl DateTime {
    /// Create a new `DateTime` instance with validation.
    ///
    /// # Errors
    ///
    /// Returns a `DateTimeError` if any component is out of valid range.
    pub fn new(
        year: u16,
        month: u8,
        day_of_month: u8,
        hour: u8,
        minute: u8,
        second: u8,
    ) -> Result<Self, DateTimeError> {
        let dt = DateTime {
            year,
            month,
            day_of_month,
            hour,
            minute,
            second,
        };
        dt.validate()?;
        Ok(dt)
    }

    /// Validate all datetime components.
    ///
    /// # Errors
    ///
    /// Returns the first `DateTimeError` encountered.
    pub fn validate(&self) -> Result<(), DateTimeError> {
        Self::validate_year(self.year)?;
        Self::validate_month(self.month)?;
        Self::validate_day(self.year, self.month, self.day_of_month)?;
        Self::validate_hour(self.hour)?;
        Self::validate_minute(self.minute)?;
        Self::validate_second(self.second)?;
        Ok(())
    }

    /// Validate the year (must be >= 1970).
    fn validate_year(year: u16) -> Result<(), DateTimeError> {
        if year < 1970 {
            return Err(DateTimeError::InvalidYear);
        }
        Ok(())
    }

    /// Validate the month (must be 1-12).
    fn validate_month(month: u8) -> Result<(), DateTimeError> {
        if month == 0 || month > 12 {
            return Err(DateTimeError::InvalidMonth);
        }
        Ok(())
    }

    /// Validate the day (must be within the valid range for the month/year).
    fn validate_day(year: u16, month: u8, day: u8) -> Result<(), DateTimeError> {
        let max_day = days_in_month(year, month);
        if day == 0 || day > max_day {
            return Err(DateTimeError::InvalidDay);
        }
        Ok(())
    }

    /// Validate the hour (must be 0-23).
    fn validate_hour(hour: u8) -> Result<(), DateTimeError> {
        if hour > 23 {
            return Err(DateTimeError::InvalidHour);
        }
        Ok(())
    }

    /// Validate the minute (must be 0-59).
    fn validate_minute(minute: u8) -> Result<(), DateTimeError> {
        if minute > 59 {
            return Err(DateTimeError::InvalidMinute);
        }
        Ok(())
    }

    /// Validate the second (must be 0-59).
    fn validate_second(second: u8) -> Result<(), DateTimeError> {
        if second > 59 {
            return Err(DateTimeError::InvalidSecond);
        }
        Ok(())
    }

    /// Get the year (e.g. 2025).
    pub fn year(&self) -> u16 {
        self.year
    }

    /// Get the month number (1-12).
    pub fn month(&self) -> u8 {
        self.month
    }

    /// Get the day of the month (1-31).
    pub fn day_of_month(&self) -> u8 {
        self.day_of_month
    }

    /// Get the hour (0-23).
    pub fn hour(&self) -> u8 {
        self.hour
    }

    /// Get the minute (0-59).
    pub fn minute(&self) -> u8 {
        self.minute
    }

    /// Get the second (0-59).
    pub fn second(&self) -> u8 {
        self.second
    }

    /// Set year with validation.
    ///
    /// Re-validates the day in case of leap-year or February issues.
    pub fn set_year(&mut self, year: u16) -> Result<(), DateTimeError> {
        Self::validate_year(year)?;
        Self::validate_day(year, self.month, self.day_of_month)?;
        self.year = year;
        Ok(())
    }

    /// Set month with validation.
    ///
    /// Re-validates the day in case month/day mismatch occurs.
    pub fn set_month(&mut self, month: u8) -> Result<(), DateTimeError> {
        Self::validate_month(month)?;
        Self::validate_day(self.year, month, self.day_of_month)?;
        self.month = month;
        Ok(())
    }

    /// Set day with validation.
    pub fn set_day_of_month(&mut self, day_of_month: u8) -> Result<(), DateTimeError> {
        Self::validate_day(self.year, self.month, day_of_month)?;
        self.day_of_month = day_of_month;
        Ok(())
    }

    /// Set hour with validation.
    pub fn set_hour(&mut self, hour: u8) -> Result<(), DateTimeError> {
        Self::validate_hour(hour)?;
        self.hour = hour;
        Ok(())
    }

    /// Set minute with validation.
    pub fn set_minute(&mut self, minute: u8) -> Result<(), DateTimeError> {
        Self::validate_minute(minute)?;
        self.minute = minute;
        Ok(())
    }

    /// Set second with validation.
    pub fn set_second(&mut self, second: u8) -> Result<(), DateTimeError> {
        Self::validate_second(second)?;
        self.second = second;
        Ok(())
    }

    /// Calculate weekday for this DateTime
    pub fn calculate_weekday(&self) -> Result<Weekday, DateTimeError> {
        calculate_weekday(self.year, self.month, self.day_of_month)
    }
}

/// Day of the week (1 = Sunday .. 7 = Saturday)
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(u8)]
pub enum Weekday {
    /// Sunday starts with 1
    Sunday = 1,
    /// Monday
    Monday = 2,
    /// Tuesday
    Tuesday = 3,
    /// Wednesday
    Wednesday = 4,
    /// Thursday
    Thursday = 5,
    /// Friday
    Friday = 6,
    /// Saturday
    Saturday = 7,
}

impl Weekday {
    /// Create a Weekday from a raw u8 (1 = Sunday .. 7 = Saturday).
    pub fn from_number(n: u8) -> Result<Self, DateTimeError> {
        match n {
            1 => Ok(Self::Sunday),
            2 => Ok(Self::Monday),
            3 => Ok(Self::Tuesday),
            4 => Ok(Self::Wednesday),
            5 => Ok(Self::Thursday),
            6 => Ok(Self::Friday),
            7 => Ok(Self::Saturday),
            _ => Err(DateTimeError::InvalidWeekday),
        }
    }

    /// Get the number form (1 = Sunday .. 7 = Saturday).
    pub fn to_number(self) -> u8 {
        self as u8
    }

    /// Get the weekday name as a string slice
    pub fn as_str(&self) -> &'static str {
        match self {
            Weekday::Sunday => "Sunday",
            Weekday::Monday => "Monday",
            Weekday::Tuesday => "Tuesday",
            Weekday::Wednesday => "Wednesday",
            Weekday::Thursday => "Thursday",
            Weekday::Friday => "Friday",
            Weekday::Saturday => "Saturday",
        }
    }
}

/// Check if a year is a leap year
pub fn is_leap_year(year: u16) -> bool {
    (year % 4 == 0) && (year % 100 != 0 || year % 400 == 0)
}

/// Get the number of days in a month
pub fn days_in_month(year: u16, month: u8) -> u8 {
    match month {
        1 | 3 | 5 | 7 | 8 | 10 | 12 => 31,
        4 | 6 | 9 | 11 => 30,
        2 => {
            if is_leap_year(year) {
                29
            } else {
                28
            }
        }
        _ => 0,
    }
}

/// Calculate the day of the week using Zeller's congruence algorithm
/// Returns 1=Sunday, 2=Monday, ..., 7=Saturday
pub fn calculate_weekday(year: u16, month: u8, day_of_month: u8) -> Result<Weekday, DateTimeError> {
    let (year, month) = if month < 3 {
        (year - 1, month + 12)
    } else {
        (year, month)
    };

    let k = year % 100;
    let j = year / 100;

    let h =
        (day_of_month as u16 + ((13 * (month as u16 + 1)) / 5) + k + (k / 4) + (j / 4) - 2 * j) % 7;

    // Convert Zeller's result (0=Saturday) to our format (1=Sunday)
    let weekday_num = ((h + 6) % 7) + 1;

    // This should never fail since we're calculating a valid weekday
    Weekday::from_number(weekday_num as u8)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_valid_datetime_creation() {
        let dt = DateTime::new(2024, 3, 15, 14, 30, 45).unwrap();
        assert_eq!(dt.year(), 2024);
        assert_eq!(dt.month(), 3);
        assert_eq!(dt.day_of_month(), 15);
        assert_eq!(dt.hour(), 14);
        assert_eq!(dt.minute(), 30);
        assert_eq!(dt.second(), 45);
    }

    #[test]
    fn test_invalid_year() {
        let result = DateTime::new(1969, 1, 1, 0, 0, 0);
        assert_eq!(result.unwrap_err(), DateTimeError::InvalidYear);
    }

    #[test]
    fn test_invalid_month() {
        assert_eq!(
            DateTime::new(2024, 0, 1, 0, 0, 0).unwrap_err(),
            DateTimeError::InvalidMonth
        );
        assert_eq!(
            DateTime::new(2024, 13, 1, 0, 0, 0).unwrap_err(),
            DateTimeError::InvalidMonth
        );
    }

    #[test]
    fn test_invalid_day() {
        // Test February 30th (invalid)
        assert_eq!(
            DateTime::new(2024, 2, 30, 0, 0, 0).unwrap_err(),
            DateTimeError::InvalidDay
        );

        // Test day 0
        assert_eq!(
            DateTime::new(2024, 1, 0, 0, 0, 0).unwrap_err(),
            DateTimeError::InvalidDay
        );

        // Test April 31st (invalid - April has 30 days)
        assert_eq!(
            DateTime::new(2024, 4, 31, 0, 0, 0).unwrap_err(),
            DateTimeError::InvalidDay
        );
    }

    #[test]
    fn test_invalid_hour() {
        assert_eq!(
            DateTime::new(2024, 1, 1, 24, 0, 0).unwrap_err(),
            DateTimeError::InvalidHour
        );
    }

    #[test]
    fn test_invalid_minute() {
        assert_eq!(
            DateTime::new(2024, 1, 1, 0, 60, 0).unwrap_err(),
            DateTimeError::InvalidMinute
        );
    }

    #[test]
    fn test_invalid_second() {
        assert_eq!(
            DateTime::new(2024, 1, 1, 0, 0, 60).unwrap_err(),
            DateTimeError::InvalidSecond
        );
    }

    #[test]
    fn test_leap_year_february_29() {
        // 2024 is a leap year - February 29th should be valid
        assert!(DateTime::new(2024, 2, 29, 0, 0, 0).is_ok());

        // 2023 is not a leap year - February 29th should be invalid
        assert_eq!(
            DateTime::new(2023, 2, 29, 0, 0, 0).unwrap_err(),
            DateTimeError::InvalidDay
        );
    }

    #[test]
    fn test_setters_with_validation() {
        let mut dt = DateTime::new(2024, 1, 1, 0, 0, 0).unwrap();

        // Valid operations
        assert!(dt.set_year(2025).is_ok());
        assert_eq!(dt.year(), 2025);

        assert!(dt.set_month(12).is_ok());
        assert_eq!(dt.month(), 12);

        assert!(dt.set_hour(23).is_ok());
        assert_eq!(dt.hour(), 23);

        // Invalid operations
        assert_eq!(dt.set_year(1969), Err(DateTimeError::InvalidYear));
        assert_eq!(dt.set_month(13), Err(DateTimeError::InvalidMonth));
        assert_eq!(dt.set_hour(24), Err(DateTimeError::InvalidHour));
    }

    #[test]
    fn test_leap_year_edge_cases_in_setters() {
        let mut dt = DateTime::new(2024, 2, 29, 0, 0, 0).unwrap(); // Leap year

        // Changing to non-leap year should fail because Feb 29 becomes invalid
        assert_eq!(dt.set_year(2023), Err(DateTimeError::InvalidDay));

        // Original value should remain unchanged after failed operation
        assert_eq!(dt.year(), 2024);
        assert_eq!(dt.day_of_month(), 29);
    }

    #[test]
    fn test_month_day_validation_in_setters() {
        let mut dt = DateTime::new(2024, 1, 31, 0, 0, 0).unwrap(); // January 31st

        // Changing to February should fail because Feb doesn't have 31 days
        assert_eq!(dt.set_month(2), Err(DateTimeError::InvalidDay));

        // Original value should remain unchanged
        assert_eq!(dt.month(), 1);
        assert_eq!(dt.day_of_month(), 31);

        // But changing to March should work (March has 31 days)
        assert!(dt.set_month(3).is_ok());
        assert_eq!(dt.month(), 3);
    }

    #[test]
    fn test_weekday_calculation() {
        let dt = DateTime::new(2024, 1, 1, 0, 0, 0).unwrap(); // New Year 2024
        let weekday = dt.calculate_weekday().unwrap();
        assert_eq!(weekday, Weekday::Monday); // January 1, 2024 was a Monday

        let dt = DateTime::new(2024, 12, 25, 0, 0, 0).unwrap();
        let weekday = dt.calculate_weekday().unwrap();
        assert_eq!(weekday, Weekday::Wednesday); // December 25, 2024 is a Wednesday
    }

    #[test]
    fn test_weekday_from_number() {
        assert_eq!(Weekday::from_number(1).unwrap(), Weekday::Sunday);
        assert_eq!(Weekday::from_number(2).unwrap(), Weekday::Monday);
        assert_eq!(Weekday::from_number(7).unwrap(), Weekday::Saturday);
        assert_eq!(Weekday::from_number(3).unwrap(), Weekday::Tuesday);

        assert_eq!(
            Weekday::from_number(0).unwrap_err(),
            DateTimeError::InvalidWeekday
        );
        assert_eq!(
            Weekday::from_number(8).unwrap_err(),
            DateTimeError::InvalidWeekday
        );
    }

    #[test]
    fn test_weekday_to_number() {
        assert_eq!(Weekday::Sunday.to_number(), 1);
        assert_eq!(Weekday::Monday.to_number(), 2);
        assert_eq!(Weekday::Saturday.to_number(), 7);
    }

    #[test]
    fn test_weekday_as_str() {
        assert_eq!(Weekday::Sunday.as_str(), "Sunday");
        assert_eq!(Weekday::Monday.as_str(), "Monday");
        assert_eq!(Weekday::Tuesday.as_str(), "Tuesday");
        assert_eq!(Weekday::Wednesday.as_str(), "Wednesday");
        assert_eq!(Weekday::Thursday.as_str(), "Thursday");
        assert_eq!(Weekday::Friday.as_str(), "Friday");
        assert_eq!(Weekday::Saturday.as_str(), "Saturday");
    }

    #[test]
    fn test_calculate_weekday_known_dates() {
        // Test some known dates
        assert_eq!(calculate_weekday(2000, 1, 1).unwrap(), Weekday::Saturday);
        assert_eq!(calculate_weekday(2024, 1, 1).unwrap(), Weekday::Monday);
        assert_eq!(calculate_weekday(2025, 8, 15).unwrap(), Weekday::Friday);

        // Test leap year boundary
        assert_eq!(calculate_weekday(2024, 2, 29).unwrap(), Weekday::Thursday); // Leap day 2024
    }

    #[test]
    fn test_is_leap_year() {
        // Regular leap years (divisible by 4)
        assert!(is_leap_year(2024));
        assert!(is_leap_year(2020));
        assert!(is_leap_year(1996));

        // Non-leap years
        assert!(!is_leap_year(2023));
        assert!(!is_leap_year(2021));
        assert!(!is_leap_year(1999));

        // Century years (divisible by 100 but not 400)
        assert!(!is_leap_year(1900));
        assert!(!is_leap_year(2100));

        // Century years divisible by 400
        assert!(is_leap_year(2000));
        assert!(is_leap_year(1600));
    }

    #[test]
    fn test_days_in_month() {
        // January (31 days)
        assert_eq!(days_in_month(2024, 1), 31);

        // February leap year (29 days)
        assert_eq!(days_in_month(2024, 2), 29);

        // February non-leap year (28 days)
        assert_eq!(days_in_month(2023, 2), 28);

        // April (30 days)
        assert_eq!(days_in_month(2024, 4), 30);

        // December (31 days)
        assert_eq!(days_in_month(2024, 12), 31);

        // Invalid month
        assert_eq!(days_in_month(2024, 13), 0);
        assert_eq!(days_in_month(2024, 0), 0);
    }

    #[test]
    fn test_setter_interdependency_edge_cases() {
        // January 31 → February (invalid because Feb max is 28/29)
        let mut dt = DateTime::new(2023, 1, 31, 0, 0, 0).unwrap();
        assert_eq!(dt.set_month(2), Err(DateTimeError::InvalidDay));

        // March 31 → April (invalid because April max is 30)
        let mut dt = DateTime::new(2023, 3, 31, 0, 0, 0).unwrap();
        assert_eq!(dt.set_month(4), Err(DateTimeError::InvalidDay));

        // Leap year Feb 29 → non-leap year
        let mut dt = DateTime::new(2024, 2, 29, 0, 0, 0).unwrap();
        assert_eq!(dt.set_year(2023), Err(DateTimeError::InvalidDay));

        // Non-leap year Feb 28 → leap year (should work)
        let mut dt = DateTime::new(2023, 2, 28, 0, 0, 0).unwrap();
        assert!(dt.set_year(2024).is_ok());
    }

    #[test]
    fn test_display_datetime_error() {
        assert_eq!(format!("{}", DateTimeError::InvalidMonth), "invalid month");
        assert_eq!(format!("{}", DateTimeError::InvalidDay), "invalid day");
        assert_eq!(format!("{}", DateTimeError::InvalidHour), "invalid hour");
        assert_eq!(
            format!("{}", DateTimeError::InvalidMinute),
            "invalid minute"
        );
        assert_eq!(
            format!("{}", DateTimeError::InvalidSecond),
            "invalid second"
        );
        assert_eq!(
            format!("{}", DateTimeError::InvalidWeekday),
            "invalid weekday"
        );
        assert_eq!(format!("{}", DateTimeError::InvalidYear), "invalid year");
    }

    #[test]
    fn test_datetime_error_trait() {
        let error = DateTimeError::InvalidMonth;
        let _: &dyn core::error::Error = &error;
    }

    #[test]
    fn test_boundary_values() {
        // Test minimum valid year
        assert!(DateTime::new(1970, 1, 1, 0, 0, 0).is_ok());

        // Test maximum valid time values
        assert!(DateTime::new(2024, 12, 31, 23, 59, 59).is_ok());

        // Test minimum valid day/month
        assert!(DateTime::new(2024, 1, 1, 0, 0, 0).is_ok());
    }

    #[test]
    fn test_february_edge_cases() {
        // Test February 28 in leap year
        assert!(DateTime::new(2024, 2, 28, 0, 0, 0).is_ok());

        // Test February 28 in non-leap year
        assert!(DateTime::new(2023, 2, 28, 0, 0, 0).is_ok());

        // Test February 29 in non-leap year (should fail)
        assert_eq!(
            DateTime::new(2023, 2, 29, 0, 0, 0).unwrap_err(),
            DateTimeError::InvalidDay
        );
    }

    #[test]
    fn test_all_month_max_days() {
        let year = 2023; // Non-leap year

        // 31-day months
        for month in [1, 3, 5, 7, 8, 10, 12] {
            assert!(DateTime::new(year, month, 31, 0, 0, 0).is_ok());
            assert_eq!(
                DateTime::new(year, month, 32, 0, 0, 0).unwrap_err(),
                DateTimeError::InvalidDay
            );
        }

        // 30-day months
        for month in [4, 6, 9, 11] {
            assert!(DateTime::new(year, month, 30, 0, 0, 0).is_ok());
            assert_eq!(
                DateTime::new(year, month, 31, 0, 0, 0).unwrap_err(),
                DateTimeError::InvalidDay
            );
        }
    }

    #[test]
    fn test_set_day_of_month() {
        let mut dt = DateTime::new(2024, 5, 15, 12, 30, 45).unwrap();

        assert!(dt.set_day_of_month(10).is_ok());
        assert_eq!(dt.day_of_month, 10);
    }

    #[test]
    fn test_all_setters_preserve_state_on_error() {
        let mut dt = DateTime::new(2024, 5, 15, 12, 30, 45).unwrap();
        let original = dt;

        // Test each setter preserves state when error occurs
        assert!(dt.set_day_of_month(40).is_err());
        assert_eq!(dt, original);

        assert!(dt.set_minute(70).is_err());
        assert_eq!(dt, original);

        assert!(dt.set_second(70).is_err());
        assert_eq!(dt, original);
    }

    #[test]
    fn test_set_minute() {
        let mut dt = DateTime::new(2024, 5, 15, 12, 30, 45).unwrap();

        assert!(dt.set_minute(10).is_ok());
        assert_eq!(dt.minute, 10);
    }

    #[test]
    fn test_set_second() {
        let mut dt = DateTime::new(2024, 5, 15, 12, 30, 45).unwrap();
        assert!(dt.set_second(10).is_ok());
        assert_eq!(dt.second, 10);
    }
}
```
