# impl Rtc

Let's implement the Rtc trait from rtc-hal for the DS3231 driver. We need to implement get_datetime and set_datetime methods.

## Get DateTime

Similar to the DS1307, we'll read data from the Seconds register through the Year register in a single burst read operation. However, the DS3231 has some key differences.

- **Clock Halt Bit Difference:** We don't need to mask out bit 7 of the seconds byte. The DS1307 required bit masking to remove the clock halt bit, but in the DS3231, bit 7 is always 0

- **Century Bit Handling:** We extract the century bit from the month register. If the century bit is set, we add +1 to our base century. For example, if our base century is 20 and the century bit is set, then year 25 becomes 2125 instead of 2025

```rust
fn get_datetime(&mut self) -> Result<rtc_hal::datetime::DateTime, Self::Error> {
    // Since DS3231 allows Subsequent registers can be accessed sequentially until a STOP condition is executed
    // Read all 7 registers in one burst operation
    let mut data = [0; 7];
    self.read_register_bytes(Register::Seconds, &mut data)?;

    // Convert from BCD format and extract fields
    let second = bcd::to_decimal(data[0]);
    let minute = bcd::to_decimal(data[1]);

    // Handle both 12-hour and 24-hour modes for hours
    let raw_hour = data[2];
    let hour = if (raw_hour & 0b0100_0000) != 0 {
        // 12-hour mode
        // Extract the Hour part (4-0 bits)
        let hr = bcd::to_decimal(raw_hour & 0b0001_1111);
        // Extract the AM/PM (5th bit). if it is set, then it is PM
        let pm = (raw_hour & 0b0010_0000) != 0;

        // Convert it to 24 hour format:
        match (hr, pm) {
            (12, false) => 0,    // 12 AM = 00:xx
            (12, true) => 12,    // 12 PM = 12:xx
            (h, false) => h,     // 1-11 AM
            (h, true) => h + 12, // 1-11 PM
        }
    } else {
        // 24-hour mode
        // Extrac the hour value from 5-0 bits
        bcd::to_decimal(raw_hour & 0b0011_1111)
    };

    // let weekday = Weekday::from_number(bcd::to_decimal(data[3]))
    //     .map_err(crate::error::Error::DateTime)?;

    let day_of_month = bcd::to_decimal(data[4]);
    // Extract century bit
    // If it is set, then it is next century
    // Let's say base century is 20, then next century will be 21
    let is_century_bit_set = (data[5] & 0b1000_0000) != 0;
    let mut century = self.base_century;
    if is_century_bit_set {
        century += 1;
    }

    let month = bcd::to_decimal(data[5] & 0b0111_1111);

    let year = (century as u16 * 100) + bcd::to_decimal(data[6]) as u16;

    rtc_hal::datetime::DateTime::new(year, month, day_of_month, hour, minute, second)
        .map_err(crate::error::Error::DateTime)
}
```

## Set DateTime

Let's implement the set_datetime method. The logic is slightly different than DS1307 since we have the century bit handling.

We calculate the full century base (i.e., 20 × 100 = 2000) and validate that the user-provided year falls within the DS3231's representable range of 200 consecutive years.

Next, we check whether the given year belongs to the base century (2000-2099) or the next century (2100-2199). Based on this determination, we set the century bit (bit 7 of the month register): clear (0) for the base century, or set (1) for the next century.

```rust
fn set_datetime(&mut self, datetime: &rtc_hal::datetime::DateTime) -> Result<(), Self::Error> {
    let century_base = self.base_century as u16 * 100;

    // Validate year is within the current or next century
    if datetime.year() < century_base || datetime.year() > (century_base + 199) {
        return Err(crate::error::Error::DateTime(DateTimeError::InvalidYear));
    }

    let is_next_century = datetime.year() >= (century_base + 100);
    let year_2digit = if is_next_century {
        (datetime.year() - century_base - 100) as u8
    } else {
        (datetime.year() - century_base) as u8
    };

    // Prepare data array for burst write (7 registers)
    let mut data = [0u8; 8];
    data[0] = Register::Seconds.addr();

    // Seconds register (0x00)
    data[1] = bcd::from_decimal(datetime.second());

    // Minutes register (0x01)
    data[2] = bcd::from_decimal(datetime.minute());

    // Hours register (0x02) - set to 24-hour mode
    // Clear bit 6 (12/24 hour mode bit) to enable 24-hour mode
    data[3] = bcd::from_decimal(datetime.hour()) & 0b0011_1111;

    let weekday = datetime
        .calculate_weekday()
        .map_err(crate::error::Error::DateTime)?;

    // Day of week register (0x03) - 1=Sunday, 7=Saturday
    data[4] = bcd::from_decimal(weekday.to_number());

    // Day of month register (0x04)
    data[5] = bcd::from_decimal(datetime.day_of_month());

    // Month register(0x05) with century bit
    let mut month_reg = bcd::from_decimal(datetime.month());
    if is_next_century {
        month_reg |= 0b1000_0000; // Set century bit
    }
    data[6] = month_reg;

    data[7] = bcd::from_decimal(year_2digit);

    // Write all 7 registers in one burst operation
    self.write_raw_bytes(&data)?;

    Ok(())
}
```

## The full code for the datetime module (datetime.rs)

```rust
//! # DateTime Module
//!
//! This module provides an implementation of the [`Rtc`] trait for the
//! DS3231 real-time clock (RTC).

use rtc_hal::{bcd, datetime::DateTimeError, rtc::Rtc};

use crate::{Ds3231, registers::Register};

impl<I2C> Rtc for Ds3231<I2C>
where
    I2C: embedded_hal::i2c::I2c,
{
    /// Read the current date and time from the DS3231.
    fn get_datetime(&mut self) -> Result<rtc_hal::datetime::DateTime, Self::Error> {
        // Since DS3231 allows Subsequent registers can be accessed sequentially until a STOP condition is executed
        // Read all 7 registers in one burst operation
        let mut data = [0; 7];
        self.read_register_bytes(Register::Seconds, &mut data)?;

        // Convert from BCD format and extract fields
        let second = bcd::to_decimal(data[0]);
        let minute = bcd::to_decimal(data[1]);

        // Handle both 12-hour and 24-hour modes for hours
        let raw_hour = data[2];
        let hour = if (raw_hour & 0b0100_0000) != 0 {
            // 12-hour mode
            // Extract the Hour part (4-0 bits)
            let hr = bcd::to_decimal(raw_hour & 0b0001_1111);
            // Extract the AM/PM (5th bit). if it is set, then it is PM
            let pm = (raw_hour & 0b0010_0000) != 0;

            // Convert it to 24 hour format:
            match (hr, pm) {
                (12, false) => 0,    // 12 AM = 00:xx
                (12, true) => 12,    // 12 PM = 12:xx
                (h, false) => h,     // 1-11 AM
                (h, true) => h + 12, // 1-11 PM
            }
        } else {
            // 24-hour mode
            // Extrac the hour value from 5-0 bits
            bcd::to_decimal(raw_hour & 0b0011_1111)
        };

        // let weekday = Weekday::from_number(bcd::to_decimal(data[3]))
        //     .map_err(crate::error::Error::DateTime)?;

        let day_of_month = bcd::to_decimal(data[4]);
        // Extract century bit
        // If it is set, then it is next century
        // Let's say base century is 20, then next century will be 21
        let is_century_bit_set = (data[5] & 0b1000_0000) != 0;
        let mut century = self.base_century;
        if is_century_bit_set {
            century += 1;
        }

        let month = bcd::to_decimal(data[5] & 0b0111_1111);

        let year = (century as u16 * 100) + bcd::to_decimal(data[6]) as u16;

        rtc_hal::datetime::DateTime::new(year, month, day_of_month, hour, minute, second)
            .map_err(crate::error::Error::DateTime)
    }

    /// Set the current date and time in the DS3231.
    ///
    /// The DS3231 stores years as 2-digit values (00-99). This method interprets
    /// the provided year based on the configured base century and its successor.
    ///
    /// # Year Range
    ///
    /// The year must be within one of these ranges:
    /// - **Base century**: `base_century * 100` to `(base_century * 100) + 99`
    /// - **Next century**: `(base_century + 1) * 100` to `(base_century + 1) * 100 + 99`
    ///
    /// For example, with `base_century = 20`:
    /// - Allowed years: 2000-2099 (stored as 00-99, century bit = 0)
    /// - Allowed years: 2100-2199 (stored as 00-99, century bit = 1)
    /// - Rejected years: 1900-1999, 2200+
    ///
    /// # Century Bit Handling
    ///
    /// The method automatically sets the DS3231's century bit based on which
    /// century range the year falls into, avoiding the ambiguity issues with
    /// this hardware feature.
    ///
    /// # Time Format
    ///
    /// The DS3231 is configured to use 24-hour time format. The weekday is
    /// calculated from the date and stored in the day register (1=Sunday, 7=Saturday).
    ///
    /// # Arguments
    ///
    /// * `datetime` - The date and time to set
    ///
    /// # Returns
    ///
    /// Returns `Err(Error::DateTime(DateTimeError::InvalidYear))` if the year
    /// is outside the supported range.
    ///
    /// # Examples
    ///
    /// ```
    /// // With base_century = 20, you can set dates from 2000-2199
    /// let datetime = DateTime::new(2023, 12, 25, 15, 30, 0)?;
    /// rtc.set_datetime(&datetime)?;
    ///
    /// // To set dates in a different century, update base_century first
    /// let rtc = rtc.with_base_century(21)?; // Now supports 2100-2299
    /// let datetime = DateTime::new(2150, 1, 1, 0, 0, 0)?;
    /// rtc.set_datetime(&datetime)?;
    /// ```
    fn set_datetime(&mut self, datetime: &rtc_hal::datetime::DateTime) -> Result<(), Self::Error> {
        let century_base = self.base_century as u16 * 100;

        // Validate year is within the current or next century
        if datetime.year() < century_base || datetime.year() > (century_base + 199) {
            return Err(crate::error::Error::DateTime(DateTimeError::InvalidYear));
        }

        let is_next_century = datetime.year() >= (century_base + 100);
        let year_2digit = if is_next_century {
            (datetime.year() - century_base - 100) as u8
        } else {
            (datetime.year() - century_base) as u8
        };

        // Prepare data array for burst write (7 registers)
        let mut data = [0u8; 8];
        data[0] = Register::Seconds.addr();

        // Seconds register (0x00)
        data[1] = bcd::from_decimal(datetime.second());

        // Minutes register (0x01)
        data[2] = bcd::from_decimal(datetime.minute());

        // Hours register (0x02) - set to 24-hour mode
        // Clear bit 6 (12/24 hour mode bit) to enable 24-hour mode
        data[3] = bcd::from_decimal(datetime.hour()) & 0b0011_1111;

        let weekday = datetime
            .calculate_weekday()
            .map_err(crate::error::Error::DateTime)?;

        // Day of week register (0x03) - 1=Sunday, 7=Saturday
        data[4] = bcd::from_decimal(weekday.to_number());

        // Day of month register (0x04)
        data[5] = bcd::from_decimal(datetime.day_of_month());

        // Month register(0x05) with century bit
        let mut month_reg = bcd::from_decimal(datetime.month());
        if is_next_century {
            month_reg |= 0b1000_0000; // Set century bit
        }
        data[6] = month_reg;

        data[7] = bcd::from_decimal(year_2digit);

        // Write all 7 registers in one burst operation
        self.write_raw_bytes(&data)?;

        Ok(())
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use embedded_hal_mock::eh1::i2c::{Mock as I2cMock, Transaction as I2cTrans};
    use rtc_hal::datetime::DateTime;

    fn new_ds3231(i2c: I2cMock) -> Ds3231<I2cMock> {
        Ds3231::new(i2c)
    }

    #[test]
    fn test_get_datetime_24h_mode() {
        // Simulate reading: sec=0x25(25), min=0x59(59), hour=0x23(23h 24h mode),
        // day_of_week=0x04, day_of_month=0x15(15), month=0x08(August), year=0x23(2023)
        let data = [0x25, 0x59, 0x23, 0x04, 0x15, 0x08, 0x23];
        let expectations = [I2cTrans::write_read(
            0x68,
            vec![Register::Seconds.addr()],
            data.to_vec(),
        )];
        let mut ds3231 = new_ds3231(I2cMock::new(&expectations));

        let dt = ds3231.get_datetime().unwrap();
        assert_eq!(dt.second(), 25);
        assert_eq!(dt.minute(), 59);
        assert_eq!(dt.hour(), 23);

        assert_eq!(dt.day_of_month(), 15);
        assert_eq!(dt.month(), 8);

        assert_eq!(dt.year(), 2023);

        ds3231.release_i2c().done();
    }

    #[test]
    fn test_set_datetime_within_base_century() {
        let datetime = DateTime::new(2025, 8, 27, 15, 30, 45).unwrap();
        // base_century = 20, so 2000-2199 valid. 2023 fits.
        let expectations = [I2cTrans::write(
            0x68,
            vec![
                Register::Seconds.addr(),
                0x45, // sec
                0x30, // min
                0x15, // hour (24h)
                0x04, // weekday (2025-08-27 is Wednesday)
                0x27, // day
                0x8,  // month
                0x25, // year (25)
            ],
        )];

        let mut ds3231 = new_ds3231(I2cMock::new(&expectations));

        ds3231.set_datetime(&datetime).unwrap();

        ds3231.release_i2c().done();
    }

    #[test]
    fn test_set_datetime_next_century() {
        let datetime = DateTime::new(2150, 1, 1, 0, 0, 0).unwrap();
        // Expect century bit set in month register
        let expectations = [I2cTrans::write(
            0x68,
            vec![
                Register::Seconds.addr(),
                0x00, // sec
                0x00, // min
                0x00, // hour
                0x05, // weekday (Thursday 2150-01-01)
                0x01, // day
                0x81, // month with century bit
                0x50, // year (50)
            ],
        )];

        let mut ds3231 = new_ds3231(I2cMock::new(&expectations));

        ds3231.set_datetime(&datetime).unwrap();

        ds3231.release_i2c().done();
    }

    #[test]
    fn test_set_datetime_invalid_year() {
        let datetime = DateTime::new(1980, 1, 1, 0, 0, 0).unwrap();
        let mut ds3231 = new_ds3231(I2cMock::new(&[]));

        let result = ds3231.set_datetime(&datetime);
        assert!(matches!(
            result,
            Err(crate::error::Error::DateTime(DateTimeError::InvalidYear))
        ));

        ds3231.release_i2c().done();
    }

    #[test]
    fn test_get_datetime_12h_mode_am() {
        // 01:15:30 AM, January 1, 2023 (Sunday)
        let data = [
            0x30,        // seconds = 30
            0x15,        // minutes = 15
            0b0100_0001, // hour register: 12h mode, hr=1, AM
            0x01,        // weekday = Sunday
            0x01,        // day of month
            0x01,        // month = January, century=0
            0x23,        // year = 23
        ];
        let expectations = [I2cTrans::write_read(
            0x68,
            vec![Register::Seconds.addr()],
            data.to_vec(),
        )];
        let mut ds3231 = new_ds3231(I2cMock::new(&expectations));

        let dt = ds3231.get_datetime().unwrap();
        assert_eq!((dt.hour(), dt.minute(), dt.second()), (1, 15, 30));

        ds3231.release_i2c().done();
    }

    #[test]
    fn test_get_datetime_12h_mode_pm() {
        // 11:45:50 PM, December 31, 2023 (Sunday)
        let data = [
            0x50,        // seconds = 50
            0x45,        // minutes = 45
            0b0110_1011, // hour register: 12h mode, hr=11, PM
            0x01,        // weekday = Sunday
            0x31,        // day of month
            0x12,        // month = December
            0x23,        // year = 23
        ];
        let expectations = [I2cTrans::write_read(
            0x68,
            vec![Register::Seconds.addr()],
            data.to_vec(),
        )];
        let mut ds3231 = new_ds3231(I2cMock::new(&expectations));

        let dt = ds3231.get_datetime().unwrap();
        assert_eq!(dt.hour(), 23); // 11 PM -> 23h
        assert_eq!(dt.month(), 12);
        assert_eq!(dt.day_of_month(), 31);

        ds3231.release_i2c().done();
    }

    #[test]
    fn test_get_datetime_12h_mode_12am() {
        // 12:10:00 AM, Feb 1, 2023 (Wednesday)
        let data = [
            0x00,        // seconds = 0
            0x10,        // minutes = 10
            0b0101_0010, // 12h mode (bit 6=1), hr=12 (0x12), AM (bit5=0)
            0x03,        // weekday = Tuesday
            0x01,        // day of month
            0x02,        // month = Feb
            0x23,        // year = 23
        ];
        let expectations = [I2cTrans::write_read(
            0x68,
            vec![Register::Seconds.addr()],
            data.to_vec(),
        )];
        let mut ds3231 = new_ds3231(I2cMock::new(&expectations));

        let dt = ds3231.get_datetime().unwrap();
        assert_eq!(dt.hour(), 0); // 12 AM should be 0h
        assert_eq!(dt.minute(), 10);

        ds3231.release_i2c().done();
    }

    #[test]
    fn test_get_datetime_12h_mode_12pm() {
        // 12:45:00 PM, Mar 1, 2023 (Wednesday)
        let data = [
            0x00,        // seconds = 0
            0x45,        // minutes = 45
            0b0111_0010, // 12h mode, hr=12, PM bit set (bit5=1)
            0x04,        // weekday = Wednesday
            0x01,        // day of month
            0x03,        // month = Mar
            0x23,        // year = 23
        ];
        let expectations = [I2cTrans::write_read(
            0x68,
            vec![Register::Seconds.addr()],
            data.to_vec(),
        )];
        let mut ds3231 = new_ds3231(I2cMock::new(&expectations));

        let dt = ds3231.get_datetime().unwrap();
        assert_eq!(dt.hour(), 12); // 12 PM should stay 12h
        assert_eq!(dt.minute(), 45);

        ds3231.release_i2c().done();
    }
}
```
