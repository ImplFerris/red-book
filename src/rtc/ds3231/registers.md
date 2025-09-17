# DS3231 Registers

The datetime registers in the DS3231 are almost the same as those in the DS1307. Both chips store time and date information in a series of registers that you can read or write through I2C.

However, there are a few differences:

- **Oscillator Control**: The DS1307 has a Clock Halt (CH) bit in the Seconds register, while the DS3231 has an Enable Oscillator (EOSC) bit in the Control register. However, they behave differently; the DS3231's EOSC bit only affects operation when the RTC is running on battery power.

- **Century bit:** Month Register (0x05) includes a century bit (bit 7) for year overflow tracking. The DS3231 can store years from 00 to 99 in its year register (like 24 for 2024). When the year rolls over from 99 back to 00 (like going from 2099 to 2100), the century bit automatically flips from 0 to 1. This tells your program that you've moved into the next century. Your code can check this bit to know whether year "24" means 2024 or 2124. This allows the DS3231 to track dates from 2000 to 2199, giving it 200 years of date range compared to the DS1307's 100 years (2000-2099).

- **No NVRAM: The DS3231 does not have the built-in NVRAM that the DS1307 offers.

- Extra features: The DS3231 includes additional registers for features the DS1307 doesn't have, such as two alarm registers, temperature registers, and control and status registers for managing the square wave output and alarm events.

## Register Map

Here is the simplified register map tables.

### DateTime Registers (0x00-0x06)

These registers store the current date and time information. They work almost the same as DS1307 registers, using BCD format for most values.

| Address | Register | Range | Notes |
|---------|----------|-------|-------|
| 0x00 | Seconds | 00-59 | Bit 7 unused (always 0) |
| 0x01 | Minutes | 00-59 | - |
| 0x02 | Hours | 01-12 or 00-23 | Bit 6: 12/24 hour mode |
| 0x03 | Day of Week | 1-7 | User defined |
| 0x04 | Date | 01-31 | - |
| 0x05 | Month/Century | 01-12 | Bit 7: Century bit |
| 0x06 | Year | 00-99 | - |


As you can see, we should be able to reuse the datetime parsing logic from DS1307 driver code with slight modifications.

### Control and Status Registers

These registers manage the DS3231's special features and show its current status.

| Address | Register | Purpose |
|---------|----------|---------|
| 0x0E | Control | Alarm enables, square wave control |
| 0x0F | Status | Alarm flags, oscillator status |
| 0x10 | Aging Offset | Crystal aging compensation |
| 0x11-0x12 | Temperature | Internal temperature sensor data |

### Alarm Registers

The DS3231 includes two programmable alarms:

- **Alarm 1 (0x07-0x0A)**: Seconds, minutes, hours, day/date precision
- **Alarm 2 (0x0B-0x0D)**: Minutes, hours, day/date precision (no seconds)
