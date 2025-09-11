# Control: Special control register

The DS1307 has a special pin called SQW/OUT that can do two different things depending on how you configure it. Think of it like a multi-purpose output that you can control through the Control Register (located at address 07h).

## What Can the SQW/OUT Pin Do?

- Option 1: Static Output - Act like a simple on/off switch

- Option 2: Square Wave Generator - Produce a repeating pulse signal at various frequencies

## Understanding the Control Register Layout

The control register is 8 bits wide, but only certain bits actually do something. Here's how the bits are arranged:

| Bit Position | 7              | 6        | 5        | 4                                    | 3        | 2        | 1           | 0           |
| ------------ | -------------- | -------- | -------- | ------------------------------------ | -------- | -------- | ----------- | ----------- |
| Bit Name     | OUT            | Always 0 | Always 0 | SQWE                                 | Always 0 | Always 0 | RS1         | RS0         |
| Description  | Output control | Unused   | Unused   | Square wave enable (1 = on, 0 = off) | Unused   | Unused   | Rate select | Rate select |


Bits 6, 5, 3, and 2 are always zero and serve no function, so you can ignore them completely.

### Bit 7 - OUT (Output Control)

This bit controls what the SQW/OUT pin outputs when the square wave generator is turned off. When SQWE equals 0 (meaning square wave is disabled), the OUT bit directly controls the pin's voltage level. 

If OUT equals 1, the SQW/OUT pin outputs HIGH voltage (typically 3.3V or 5V). If OUT equals 0, the pin outputs LOW voltage (0V). You might use this to turn an LED on or off, or to provide a control signal to other circuits in your project.

### Bit 4 - SQWE (Square Wave Enable)

Think of this bit as the master power switch for square wave generation. When SQWE equals 1, the square wave generator turns on and the pin outputs the frequency you've selected with the rate select bits. When SQWE equals 0, the square wave generator is completely off and the pin simply follows whatever you've set the OUT bit to. This bit gives you complete control over whether you want static output or a pulsing signal.

### Bits 1-0 - RS1, RS0 (Rate Select)

These two bits work together to choose the square wave frequency, but they only matter when SQWE equals 1. The DS1307 offers four different frequency options based on how you set these bits.

| RS1 | RS0 | Frequency  |
| --- | --- | ---------- |
| 0   | 0   | 1 Hz       |
| 0   | 1   | 4.096 kHz  |
| 1   | 0   | 8.192 kHz  |
| 1   | 1   | 32.768 kHz |

When both RS1 and RS0 equal 0, you get 1Hz, which means one complete pulse every second. This frequency works perfectly for creating a visible blink on an LED or generating a second tick for clock displays.

Setting RS1 to 0 and RS0 to 1 gives you 4.096kHz, which completes about 4,096 pulses per second. This faster frequency suits audio applications or situations where you need moderate-speed timing signals.

When RS1 equals 1 and RS0 equals 0, the output becomes 8.192kHz for even faster timing applications that need quick pulses.

Finally, setting both RS1 and RS0 to 1 produces 32.768kHz, which matches the crystal frequency inside the DS1307. This very fast signal works well when you need to provide a high-speed clock signal to other microcontrollers or digital circuits.

## Practical Examples

### Example 1: Simple LED Control

Suppose you want to turn an LED on and leave it on. You would set the control register to binary 10000000, which equals 0x80 in hexadecimal. This sets OUT to 1 (turning the LED on), SQWE to 0 (no square wave needed), and the RS bits don't matter since the square wave is disabled.

### Example 2: 1Hz Blinking LED

For a LED that blinks once per second, set the control register to binary 00010000, which equals 0x10 in hexadecimal. The OUT bit doesn't matter because SQWE overrides it when enabled. SQWE equals 1 to enable square wave generation, and both RS1 and RS0 equal 0 to select 1Hz frequency.

### Example 3: High-Speed Clock Signal

To generate a 32.768kHz clock signal for another circuit, use binary 00010011, which equals 0x13 in hexadecimal. SQWE equals 1 to enable the square wave, and both RS1 and RS0 equal 1 to select the fastest available frequency.

## Power-On Behavior

When you first power up the DS1307, the control register starts with specific default values. OUT begins at 0, making the SQW/OUT pin start in the LOW state. SQWE also starts at 0, meaning square wave generation is disabled initially. The RS1 and RS0 bits default to 1 and 1 respectively, setting the fastest frequency, but since SQWE starts disabled, this frequency setting doesn't take effect until you enable square wave mode.

## Why Use Each Mode?

Static mode (when SQWE equals 0) useful for controlling LEDs, and other devices that need simple on/off control. You can also use it to provide enable or disable signals to other circuits in your project.

Square wave mode (when SQWE equals 1) is for generating clock signals for other microcontrollers, creating precise timing references, driving buzzer circuits for alarms, or synchronizing multiple systems in larger projects.
