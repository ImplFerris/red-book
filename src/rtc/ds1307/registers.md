# DS1307 Registers

The DS1307 contains two main register areas. The RTC registers are found at addresses 00h to 07h. These store the current time, date, and control settings. The onboard RAM is located at addresses 08h to 3Fh and can store up to 56 bytes of user data.

When you perform a multibyte read or write, the internal address pointer automatically moves to the next location. If it reaches 3Fh (the last RAM address), it wraps back to 00h, which is the start of the clock registers.

## Register Map

Here is the simplified register map table. The datasheet shows one big table with address and all the bit details together. I split it up to make it easier to understand. This first table shows what each register address does. Each register in the DS1307 stores 8 bits of data.

| ADDRESS | FUNCTION |
|---------|----------|
| 00h | Seconds |
| 01h | Minutes |
| 02h | Hours |
| 03h | Day |
| 04h | Date |
| 05h | Month |
| 06h | Year |
| 07h | Control |
| 08h-3Fh | RAM |

I will explain the calendar-related registers and the control register in depth in the next chapter. For now, we will give a quick overview of the RAM registers.

### RAM: 56 bytes of non-volatile storage

The DS1307 gives you 56 bytes of RAM to store your own data. This memory is non-volatile, meaning it retains its contents even when the device is powered down, as long as a backup battery is connected.

The RAM uses addresses 08h to 3Fh (that's 56 locations total). Each location can store one byte of data, from 00h to FFh.

```sh
Addresses: 08h, 09h, 0Ah ... 3Eh, 3Fh
Storage:   56 bytes total
Data:      Any value from 00h to FFh
```

You can store any data you want here - numbers, settings, flags, or small data logs. Unlike the time registers that need special BCD format, RAM accepts any value from 00h to FFh directly.

**Example uses**

Store user settings like alarm time, temperature readings from sensors, or flags to remember what your device was doing. With a backup battery, your device remembers this data when it starts up again.
