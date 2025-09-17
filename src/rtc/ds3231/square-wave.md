# DS3231 Square Wave Output Control

Unlike the DS1307, the DS3231 has a dedicated 32kHz output pin separate from its INT/SQW pin. The INT/SQW pin functions similarly to the DS1307's SQW/OUT pin but with limited frequency options, supporting square wave outputs only up to 8.192kHz (1Hz, 1.024kHz, 4.096kHz, and 8.192kHz).

For applications requiring a 32kHz square wave, the DS3231's separate 32kHz pin must be used, as this frequency is not available on the INT/SQW pin.

When implementing the RTC HAL Square Wave trait, we will focus exclusively on controlling the INT/SQW pin. The dedicated 32kHz pin will not be managed through this trait implementation to maintain clear separation of functionality and avoid confusion between the two distinct output pins. This approach ensures the Square Wave trait remains focused on the configurable frequency outputs available through the INT/SQW pin.

In this chapter, we will look into the control register (0x0E). 

| Bit | Name  | POR | Description |
|-----|-------|-----|-------------|
| 7   | <span style="text-decoration: overline;">EOSC</span>  | 0   | Enable Oscillator (0=enabled, 1=disabled when on battery) |
| 6   | BBSQW | 0   | Battery-Backed Square Wave Enable (0=disabled, 1=enabled) |
| 5   | CONV  | 0   | Convert Temperature (1=manual trigger) |
| 4   | RS2   | 1   | Rate Select bit 2 (square wave frequency select) |
| 3   | RS1   | 1   | Rate Select bit 1 (square wave frequency select) |
| 2   | INTCN | 1   | Interrupt Control (0=square wave, 1=interrupt mode) |
| 1   | A2IE  | 0   | Alarm 2 Interrupt Enable |
| 0   | A1IE  | 0   | Alarm 1 Interrupt Enable |

POR = Power-On Reset (default values after power-up or reset)

The important bits related to square wave control are INTCN (Bit 2) which must be cleared to 0 to enable square wave mode (default is 1 for interrupt mode), RS2 (Bit 4) and RS1 (Bit 3) which are the Rate Select bits that control frequency output, and BBSQW (Bit 6) which enables square wave output to continue on battery power when set to 1 (default is 0 for disabled). We won't focus on the other bits right now.

### Squware Wave Frequency

Setting the square wave frequency is similar to DS1307, where you configure the Rate Select bits (RS1 and RS2) in the Control Register to choose from the available frequency options. The DS3231 offers four frequency settings: 1Hz, 1.024kHz, 4.096kHz, and 8.192kHz.  To enable square wave output, the INTCN bit must be cleared to 0, switching the device from interrupt mode to square wave mode.

| RS2 | RS1 | Frequency |
|-----|-----|-----------|
| 0   | 0   | 1 Hz      |
| 0   | 1   | 1.024 kHz |
| 1   | 0   | 4.096 kHz |
| 1   | 1   | 8.192 kHz |

As you can see, the DS3231 includes 1.024 kHz also as one of its frequency options.
