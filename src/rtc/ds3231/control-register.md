# Control Register

The DS3231 contains two main control registers that configure device operation, alarm functionality, and output settings. These registers are accessed via I2C at addresses 0x0E and 0x0F.

## Control Register (0x0E)

The Control Register allow us to configure the basic operation of the DS3231, including oscillator settings, alarm interrupt enables, and output control.

### Register Layout

| Bit | Name | Default | Description |
|-----|------|---------|-------------|
| 7 | EOSC | 0 | Enable Oscillator |
| 6 | BBSQW | 0 | Battery-Backed Square Wave Enable |
| 5 | CONV | 0 | Convert Temperature |
| 4 | RS2 | 1 | Rate Select 2 |
| 3 | RS1 | 1 | Rate Select 1 |
| 2 | INTCN | 1 | Interrupt Control |
| 1 | A2IE | 0 | Alarm 2 Interrupt Enable |
| 0 | A1IE | 0 | Alarm 1 Interrupt Enable |

### Bit Descriptions

**EOSC (Bit 7) - Enable Oscillator**

If we set it to 1, the oscillator stops but only when running on battery power (it still runs normally on main power).  If we set it to 0, the oscillator keeps running. This is when you want to save battery life by stopping timekeeping when the main power is off.

**BBSQW (Bit 6) - Battery-Backed Square Wave Enable**

This controls whether the square wave output continues operating when the device is running on battery power. The default setting of 0 disables the square wave output during battery operation to save power. Setting it to 1 keeps the square wave active even on battery power. This setting only matters when the INTCN bit is set to 0.

**CONV (Bit 5) - Convert Temperature**

The DS3231 has a built-in temperature sensor that normally takes readings automatically. Setting this bit to 1 forces an immediate temperature conversion. The bit automatically clears itself when the conversion completes.

**RS2, RS1 (Bits 4-3) - Rate Select**

These two bits work together to set the frequency of the square wave output when INTCN is set to 0. The default setting produces an 8.192 kHz signal, but you can configure it for 1 Hz, 1.024 kHz, or 4.096 kHz depending on your application needs.

Square wave output frequency selection when INTCN = 0:

| RS2 | RS1 | Frequency |
|-----|-----|-----------|
| 0 | 0 | 1 Hz |
| 0 | 1 | 1.024 kHz |
| 1 | 0 | 4.096 kHz |
| 1 | 1 | 8.192 kHz (default) |

**INTCN (Bit 2) - Interrupt Control**

This is a key bit that determines how the SQW/INT pin behaves. When set to 1 (the default), the pin acts as an interrupt output for alarms. When set to 0, the pin outputs a square wave at the frequency determined by the RS bits. You cannot have both interrupts and square wave output simultaneously.

**A2IE and A1IE (Bits 1-0) - Alarm Interrupt Enable**

These bits enable or disable interrupt generation for Alarm 2 and Alarm 1 respectively. Both default to 0 (disabled). For the interrupts to actually work, you also need to set INTCN to 1. When an alarm triggers, the corresponding alarm flag in the status register gets set, and if interrupts are enabled, the SQW/INT pin will go low.

## Control/Status Register (0x0F)

The Control/Status Register provides status information and additional control functions.  We will not be implementing these register in our chapter for now.

### Register Layout

| Bit | Name | Default | Description |
|-----|------|---------|-------------|
| 7 | OSF | 1 | Oscillator Stop Flag |
| 6 | 0 | 0 | Reserved (always 0) |
| 5 | 0 | 0 | Reserved (always 0) |
| 4 | 0 | 0 | Reserved (always 0) |
| 3 | EN32kHz | 1 | Enable 32kHz Output |
| 2 | BSY | 0 | Busy |
| 1 | A2F | 0 | Alarm 2 Flag |
| 0 | A1F | 0 | Alarm 1 Flag |

### Bit Descriptions

**OSF (Bit 7) - Oscillator Stop Flag**

This bit gets set to 1 whenever the oscillator stops running. This includes when you first power up the DS3231, when there isn't enough voltage on VCC or VBAT, or if the oscillator stops for any other reason. The bit stays at 1 until you write 0 to clear it. This bit does not affect or control the oscillator - it's only a status indicator.

**EN32kHz (Bit 3) - Enable 32kHz Output**

The DS3231 has a separate 32kHz output pin that can provide a precise 32.768 kHz square wave signal. This bit controls whether that output is active. The default setting of 1 enables the output, while setting it to 0 disables it to save power.

**BSY (Bit 2) - Busy**

This read-only bit indicates when the DS3231 is busy performing a temperature conversion. During the conversion process, this bit will be set to 1. You can check this bit to know when a temperature reading is complete.

**A2F and A1F (Bits 1-0) - Alarm Flags**

These are status flags that get set to 1 when their corresponding alarms trigger. When Alarm 1 or Alarm 2 matches the current time, the DS3231 automatically sets the appropriate flag. These flags remain set until you manually clear them by writing 0 to them.
