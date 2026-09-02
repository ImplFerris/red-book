{{#title DS3231 RTC Alarm Register Explained}}

# DS3231 Alarm

We discussed some of the alarm-related bits in the [control register section](./control-register.md) of the DS3231 chapter. However, we have not looked at the alarm registers in detail yet.

Before getting into the implementation of the `Alarm` trait, let's first understand the alarm registers and how alarms work in the DS3231.

<style>
  .styled-table {
    width: 100%;
    border-collapse: collapse;
    font-family: "Roboto", Arial, sans-serif;
    font-size: 14px;
    text-align: center;
    border-radius: 6px;
    overflow: hidden;
    box-shadow: 0 2px 6px rgba(0,0,0,0.15);
  }

  .styled-table th {
    background-color: #1c1c1c;
    color: #ffffff;
    padding: 10px;
    font-weight: 600;
    border: 1px solid #424242;
  }

  .styled-table td {
    padding: 10px;
    border: 1px solid #424242;
  }

  .styled-tr:nth-child(even) {
    background-color: #2e2e2e;
  }

  .register-name {
    background-color: #3a3a3a;
    color: #ffffff;
    font-weight: 600;
    text-align: left;
    padding-left: 12px;
  }

  .special-bit {
    background-color: #bf360c;
    color: #ffffff;
    font-weight: 500;
  }

  .fixed-bit {
    background-color: #546e7a;
    color: #ffffff;
  }

  .group-tens {
    background-color: #1565c0;
    color: #ffffff;
    font-weight: 500;
  }

  .group-ones {
    background-color: #2e7d32;
    color: #ffffff;
    font-weight: 500;
  }

  .alarm-match-bit {
    background-color: #FF6F00;
    color: #ffffff;
    font-weight: 500;
  }

  .day-bit {
    background-color: #7b1fa2;
    color: #ffffff;
    font-weight: 500;
  }
</style>

## Alarm 1 Registers

Alarm 1 uses four registers from `0x07` to `0x0A`. These registers store the seconds, minutes, hours, and day/date values used for matching.

<table class="styled-table">
    <tr class="styled-tr">
        <th>REGISTER</th>
        <th>BIT 7</th>
        <th>BIT 6</th>
        <th>BIT 5</th>
        <th>BIT 4</th>
        <th>BIT 3</th>
        <th>BIT 2</th>
        <th>BIT 1</th>
        <th>BIT 0</th>
    </tr>
    <tr>
        <td class="register-name">Alarm 1 Seconds</td>
        <td class="alarm-match-bit">A1M1</td>
        <td colspan="3" class="group-tens">Tens digit of seconds</td>
        <td colspan="4" class="group-ones">Ones digit of seconds</td>
    </tr>
    <tr>
        <td class="register-name">Alarm 1 Minutes</td>
        <td class="alarm-match-bit">A1M2</td>
        <td colspan="3" class="group-tens">Tens digit of minutes</td>
        <td colspan="4" class="group-ones">Ones digit of minutes</td>
    </tr>
    <tr>
        <td class="register-name">Alarm 1 Hours</td>
        <td class="alarm-match-bit">A1M3</td>
        <td class="special-bit">Select <br/>12/24 format</td>
        <td class="special-bit">PM/AM Flag in 12 hour format<br/>or<br/>20 Hour bit in 24 hour format</td>
        <td class="group-tens">Tens digit of hours</td>
        <td colspan="4" class="group-ones">Ones digit of hours</td>
    </tr>
    <tr>
        <td class="register-name">Alarm 1 Day/Date</td>
        <td class="alarm-match-bit">A1M4</td>
        <td class="day-bit">Select Day of The Week (e.g: Sunday) <br/>or<br/> Day of Month (e.g: 31st)</td>
        <td colspan="2" class="group-tens">Tens digit of day of the month</td>
        <td colspan="4" class="group-ones">Ones digit of Day/Date</td>
    </tr>
</table>

### Alarm 1 Seconds: Range 00-59

The bits 0-3 represent the ones digit of the seconds. The bits 4-6 represent the tens digit part of the seconds.

Bit 7 is the `A1M1` mask bit. When it is `0`, the alarm is triggered when the seconds value matches the current seconds. When it is `1`, the seconds value is ignored.

If you want Alarm 1 to match seconds 30, the BCD value will be: `0011 0000`. We will put `011` in bits 4-6 and `0000` in bits 0-3.

```sh
Bit:     7 6 5 4 3 2 1 0
Value:   0 0 1 1 0 0 0 0
         │ └─┬─┘ └──┬──┘
         │   3      0
         A1M1
```

### Alarm 1 Minutes: Range 00-59

This register works in the same way as the Seconds register.

Let's say we want Alarm 1 to match minute 30. The BCD value will be `0011 0000`. We put `011` in bits 4-6 and `0000` in bits 0-3. We should also set the `A1M2` mask bit to `0`.

```sh
Bit:     7 6 5 4 3 2 1 0
Value:   0 0 1 1 0 0 0 0
         │ └─┬─┘ └──┬──┘
         │   3      0
         A1M2
```


### Alarm 1 Hours: Range 1-12 + AM/PM or 00-23

The Alarm 1 Hours register uses the same 12-hour and 24-hour formats as the main Hours register.

Bit 7 is the `A1M3` mask bit. When it is `0`, the hour value must match the current hour. When it is `1`, the hour value is ignored.

In 24-hour mode, bit 6 is `0`, bit 5 is the 20-hour bit, bit 4 is the remaining tens digit, and bits 3-0 contain the ones digit.

For example, if we want Alarm 1 to match 07:00, the hour value will be `0000 0111` and `A1M3` will be `0`.

```sh
Bit:     7 6 5 4 3 2 1 0
Value:   0 0 0 0 0 1 1 1
         │ │ └─┬─┘ └─┬─┘
         │ │   0     7
         │ 24-hour format
         A1M3
```

### Alarm 1 Day/Date

The Alarm 1 Day/Date register can be used to match either the day of the month or the day of the week.

Bit 7 is the `A1M4` mask bit. When it is `0`, the day/date value is used for matching. When it is `1`, the day/date value is ignored.

Bit 6 is the `DY/DT` bit. When it is `0`, the value represents the day of the month (i.e., the date). When it is `1`, the value represents the day of the week.

> [!NOTE]
> The datasheet refers to the day of the month as the "date of the month"

For example, to match the 15th day of the month, `DY/DT` is `0` and the date value is `15`.

```sh
Bit:     7 6 5 4 3 2 1 0
Value:   0 0 0 1 0 1 0 1
         │ │ └─┬─┘ └─┬─┘
         │ │   1     5
         │ Day of month
         A1M4
```

To match a particular weekday, `DY/DT` is set to `1` and bits 2-0 contain the weekday value.

For example, if you programmed the Day register with `1 = Sunday`, `2 = Monday`, and so on, Monday is represented by the value `2`, and the alarm's day value must match it.

```sh
Bit:     7 6 5 4 3 2 1 0
Value:   0 1 0 0 0 0 1 0
         │ │ └────┬────┘
         │ │      2
         │ Day of week
         A1M4
```

## Alarm 2 Registers

Alarm 2 uses three registers from `0x0B` to `0x0D`. Unlike Alarm 1, it does not have a seconds register.

<table class="styled-table">
    <tr class="styled-tr">
        <th>REGISTER</th>
        <th>BIT 7</th>
        <th>BIT 6</th>
        <th>BIT 5</th>
        <th>BIT 4</th>
        <th>BIT 3</th>
        <th>BIT 2</th>
        <th>BIT 1</th>
        <th>BIT 0</th>
    </tr>
    <tr>
        <td class="register-name">Alarm 2 Minutes</td>
        <td class="alarm-match-bit">A2M2</td>
        <td colspan="3" class="group-tens">Tens digit of minutes</td>
        <td colspan="4" class="group-ones">Ones digit of minutes</td>
    </tr>
    <tr>
        <td class="register-name">Alarm 2 Hours</td>
        <td class="alarm-match-bit">A2M3</td>
        <td class="special-bit">Select <br/>12/24 format</td>
        <td class="special-bit">PM/AM Flag in 12 hour format<br/>or<br/>20 Hour bit in 24 hour format</td>
        <td class="group-tens">Tens digit of hours</td>
        <td colspan="4" class="group-ones">Ones digit of hours</td>
    </tr>
    <tr>
        <td class="register-name">Alarm 2 Day/Date</td>
        <td class="alarm-match-bit">A2M4</td>
        <td class="day-bit">Select Day of The Week (e.g: Sunday) <br/>or<br/> Day of Month (e.g: 31st)</td>
        <td colspan="2" class="group-tens">Tens digit of day of the month</td>
        <td colspan="4" class="group-ones">Ones digit of Day/Date</td>
    </tr>
</table>

## Alarm Matching

The `A1Mx` and `A2Mx` bits are mask bits. When a mask bit is `0`, the corresponding field is used for matching. When it is `1`, the field is ignored.

However, these bits cannot be combined arbitrarily. The DS3231 supports only specific combinations of mask bits. Configurations that are not listed in the datasheet can result in illogical operation.

For Alarm 1, the supported configurations are:

> [!NOTE]
> Here, `X` means "don't care". The value of that bit can be either `0` or `1` because it does not affect the alarm matching for that configuration.

| DY/DT | A1M4 | A1M3 | A1M2 | A1M1 | Alarm Rate                                      |
| ----- | ---- | ---- | ---- | ---- | ----------------------------------------------- |
| X     | 1    | 1    | 1    | 1    | Once per second (nothing is required to match) |
| X     | 1    | 1    | 1    | 0    | When seconds match                              |
| X     | 1    | 1    | 0    | 0    | When minutes and seconds match                  |
| X     | 1    | 0    | 0    | 0    | When hours, minutes, and seconds match          |
| 0     | 0    | 0    | 0    | 0    | When date, hours, minutes, and seconds match    |
| 1     | 0    | 0    | 0    | 0    | When day, hours, minutes, and seconds match     |

For example, an Alarm 1 for every day at 07:30 needs the hour, minute, and second fields to match. We can set the hour to `07`, the minute to `30`, and the second to `00`, with `A1M1`, `A1M2`, and `A1M3` set to `0` and `A1M4` set to `1`.

Alarm 2 has similar restrictions, but it does not have a seconds field:

| DY/DT | A2M4 | A2M3 | A2M2 | Alarm Rate                                   |
| ----- | ---- | ---- | ---- | -------------------------------------------- |
| X     | 1    | 1    | 1    | Once per minute (at 00 seconds)              |
| X     | 1    | 1    | 0    | When minutes match                           |
| X     | 1    | 0    | 0    | When hours and minutes match                 |
| 0     | 0    | 0    | 0    | When date, hours, and minutes match          |
| 1     | 0    | 0    | 0    | When day, hours, and minutes match           |


## Alarm Behavior

When the RTC register values match the configured alarm values, the corresponding alarm flag, `A1F` or `A2F`, is set to `1`.

If the corresponding alarm interrupt enable bit, `A1IE` or `A2IE`, is also set to `1`, and the `INTCN` bit is set to `1`, the alarm condition activates the `INT/SQW` signal.

When an alarm triggers the `INT/SQW` output, the signal itself does not indicate which alarm triggered it because either alarm can activate the output. We need to check the `A1F` and `A2F` flags to determine which alarm was triggered.

> [!NOTE]
> The alarm flags `A1F` and `A2F` are in the Control/Status register (`0x0F`), while the alarm interrupt enable bits `A1IE` and `A2IE` are in the Control register (`0x0E`). Refer to the [Control Register](./control-register.md) section for more details.

The `A1F` and `A2F` flags remain set after the alarm is triggered. To clear an alarm flag, we write `0` to the corresponding flag. Writing `1` does not change the flag.

The alarm match is checked once every second when the RTC updates the time and date registers.
