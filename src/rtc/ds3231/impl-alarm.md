{{#title Implementing Alarm Feature for the DS3231 in Embedded Rust | RED Book}}

# Implementing Alarm for the DS3231

Now that we understand how the DS3231 alarm registers work and how alarm matching is performed, we can implement the `Alarm` trait for the DS3231 driver.


## Extending the Registers

The alarm registers are not part of the registers we have used so far, so we first need to add them to the `Register` enum in the register module. This includes the registers for Alarm 1, Alarm 2, and the Control and Control/Status registers.

```rust
/// Alarm 1 seconds register (0x07)
Alarm1Seconds = 0x07,

/// Alarm 1 minutes register (0x08)
Alarm1Minutes = 0x08,

/// Alarm 1 hours register (0x09)
Alarm1Hours = 0x09,

/// Alarm 1 day/date register (0x0A)
Alarm1DayDate = 0x0A,

/// Alarm 2 minutes register (0x0B)
Alarm2Minutes = 0x0B,

/// Alarm 2 hours register (0x0C)
Alarm2Hours = 0x0C,

/// Alarm 2 day/date register (0x0D)
Alarm2DayDate = 0x0D,

/// Control register (0x0E)
Control = 0x0E,

/// Control/Status register (0x0F)
ControlStatus = 0x0F,
```

## Adding Alarm Bit Masks

The alarm interrupt and status bits are located in the Control and Control/Status registers. We will define bit masks for these alarm bits so that we can easily access them without directly using their bit positions.

```rust

/// Alarm 2 Interrupt Enable - Control register (0x0E) bit
pub const A2IE_BIT: u8 = 1 << 1;

/// Alarm 1 Interrupt Enable - Control register (0x0E) bit
pub const A1IE_BIT: u8 = 1 << 0;

/// Alarm 2 Flag - Control/Status register (0x0F) bit
pub const A2F_BIT: u8 = 1 << 1;

/// Alarm 1 Flag - Control/Status register (0x0F) bit
pub const A1F_BIT: u8 = 1 << 0;

```

Now, let's create the `alarm.rs` module and start working on it.

## Defining Alarm IDs

The RTC hardware might have more than one alarm. In the RTC HAL, we already defined an associated type for the alarm ID. In the DS3231 implementation, we need to define an `AlarmId` enum and use it as the alarm ID. Since the DS3231 has two alarms, we will create two enum variants.

```rust
/// Identifies an alarm supported by the DS3231.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[cfg_attr(feature = "defmt", derive(defmt::Format))]
pub enum AlarmId {
    /// Alarm 1.
    Alarm1,
    /// Alarm 2.
    Alarm2,
}
```

## Implementing Alarm 1

Alarm 1 uses four registers to define its matching conditions: seconds, minutes, hours, and day/date. Before writing these values to the registers, we need to translate the generic `AlarmConfig` into the format expected by the DS3231.

We first perform the validation required by the DS3231. Then, we send the seconds register address along with the alarm data to the DS3231.

```rust
impl<I2C> Ds3231<I2C>
where
    I2C: embedded_hal::i2c::I2c,
{
    fn alarm_value(value: Option<u8>) -> u8 {
        match value {
            Some(value) => bcd::from_decimal(value),
            None => 1 << 7,
        }
    }

    fn set_alarm_1(&mut self, alarm: AlarmConfig) -> Result<(), <Self as ErrorType>::Error> {
        let second = alarm.second();
        let minute = alarm.minute();
        let hour = alarm.hour();
        let weekday = alarm.weekday();
        let day_of_month = alarm.day_of_month();

        if second.is_none() && minute.is_some() {
            return Err(Error::InvalidAlarmConfig(
                "Alarm 1 - minute matching requires second",
            ));
        }

        if minute.is_none() && hour.is_some() {
            return Err(Error::InvalidAlarmConfig(
                "Alarm 1 - hour matching requires minute",
            ));
        }

        if hour.is_none() && (weekday.is_some() || day_of_month.is_some()) {
            return Err(Error::InvalidAlarmConfig(
                "Alarm 1 - day matching requires hour",
            ));
        }

        let mut data = [0u8; 5];
        data[0] = Register::Alarm1Seconds.addr();
        data[1] = Self::alarm_value(second);
        data[2] = Self::alarm_value(minute);
        data[3] = Self::alarm_value(hour);

        data[4] = if weekday.is_none() && day_of_month.is_none() {
            1 << 7
        } else {
            match weekday {
                Some(weekday) => bcd::from_decimal(weekday.to_number()) | (1 << 6),
                None => bcd::from_decimal(day_of_month.unwrap_or(1)),
            }
        };

        self.write_raw_bytes(&data)?;

        Ok(())
    }
}
```

## Implementing Alarm 2

Alarm 2 works in a similar way to Alarm 1, but it does not have a seconds register. Therefore, we need to reject configurations that include seconds.

The remaining fields are written in the same way as Alarm 1.

```rust
impl<I2C> Ds3231<I2C>
where
    I2C: embedded_hal::i2c::I2c,
{
    //...
    //set_alarm_1 function ...
    //

    fn set_alarm_2(&mut self, alarm: AlarmConfig) -> Result<(), <Self as ErrorType>::Error> {
        let second = alarm.second();
        let minute = alarm.minute();
        let hour = alarm.hour();
        let weekday = alarm.weekday();
        let day_of_month = alarm.day_of_month();

        if second.is_some() {
            return Err(Error::InvalidAlarmConfig(
                "Alarm 2 does not support seconds",
            ));
        }

        if minute.is_none() && hour.is_some() {
            return Err(Error::InvalidAlarmConfig(
                "Alarm 2 - hour matching requires minute",
            ));
        }

        if hour.is_none() && (weekday.is_some() || day_of_month.is_some()) {
            return Err(Error::InvalidAlarmConfig(
                "Alarm 2 - day matching requires hour",
            ));
        }

        let mut data = [0u8; 4];
        data[0] = Register::Alarm2Minutes.addr();

        data[1] = Self::alarm_value(minute);
        data[2] = Self::alarm_value(hour);

        data[3] = if weekday.is_none() && day_of_month.is_none() {
            1 << 7
        } else {
            match weekday {
                Some(weekday) => bcd::from_decimal(weekday.to_number()) | (1 << 6),
                None => bcd::from_decimal(day_of_month.unwrap_or(1)),
            }
        };

        self.write_raw_bytes(&data)?;

        Ok(())
    }
}
```

## Implementing the `Alarm` Trait

Now that we have the functions for configuring both alarms, we can implement the RTC HAL's `Alarm` trait for the DS3231 driver.

In the `set_alarm` function, we first check the restrictions specific to the DS3231 and then call the function for the selected alarm.

In the `enable_alarm` function, we set the `INTCN` bit so that the `INT/SQW` pin is used for alarm interrupts. Then, we enable the corresponding interrupt bit for the selected alarm.

In the `disable_alarm` function, we only clear the interrupt bit for the selected alarm.

In the `alarm_triggered` function, we check the corresponding alarm flag to determine whether the alarm has been triggered. In the `clear_alarm` function, we clear the flag for the selected alarm.

```rust
impl<I2C> Alarm for Ds3231<I2C>
where
    I2C: embedded_hal::i2c::I2c,
{
    type AlarmId = AlarmId;

    fn set_alarm(&mut self, id: Self::AlarmId, alarm: AlarmConfig) -> Result<(), Self::Error> {
        if alarm.month().is_some() {
            return Err(Error::InvalidAlarmConfig(
                "DS3231 alarms do not support month matching",
            ));
        }

        if alarm.weekday().is_some() && alarm.day_of_month().is_some() {
            return Err(Error::InvalidAlarmConfig(
                "alarm cannot match both weekday and day of month",
            ));
        }

        match id {
            AlarmId::Alarm1 => self.set_alarm_1(alarm),
            AlarmId::Alarm2 => self.set_alarm_2(alarm),
        }
    }

    fn enable_alarm(&mut self, id: Self::AlarmId) -> Result<(), Self::Error> {
        let mut control = self.read_register(Register::Control)?;

        control |= registers::INTCN_BIT;

        control |= match id {
            AlarmId::Alarm1 => registers::A1IE_BIT,
            AlarmId::Alarm2 => registers::A2IE_BIT,
        };

        self.write_register(Register::Control, control)
    }

    fn disable_alarm(&mut self, id: Self::AlarmId) -> Result<(), Self::Error> {
        let mut control = self.read_register(Register::Control)?;

        control &= match id {
            AlarmId::Alarm1 => !registers::A1IE_BIT,
            AlarmId::Alarm2 => !registers::A2IE_BIT,
        };

        self.write_register(Register::Control, control)
    }

    fn alarm_triggered(&mut self, id: Self::AlarmId) -> Result<bool, Self::Error> {
        let status = self.read_register(Register::ControlStatus)?;

        let alarm_flag = match id {
            AlarmId::Alarm1 => status & registers::A1F_BIT,
            AlarmId::Alarm2 => status & registers::A2F_BIT,
        };

        Ok(alarm_flag != 0)
    }

    fn clear_alarm(&mut self, id: Self::AlarmId) -> Result<(), Self::Error> {
        let mut status = self.read_register(Register::ControlStatus)?;

        status &= match id {
            AlarmId::Alarm1 => !registers::A1F_BIT,
            AlarmId::Alarm2 => !registers::A2F_BIT,
        };

        self.write_register(Register::ControlStatus, status)
    }
}
```

## Re-exporting the Alarm Types

Finally, update `lib.rs` to re-export the alarm types so they can be used directly by applications.

```rust
// Re-export RTC HAL
pub use rtc_hal::{
    alarm::{Alarm, AlarmBuilder, AlarmConfig},
    datetime::DateTime, //already exists
    rtc::Rtc, //already exists
};
```
