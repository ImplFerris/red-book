# Real-Time Clock (RTC) Hardware Abstraction Layer (HAL)

In this chapter, we'll develop an RTC HAL crate that provides generic traits for Real Time Clock (RTC) functionality. Our design follows the embedded-hal design pattern, creating a standardized interface that can work across different RTCs.

> Note: This is just one approach to structuring an RTC HAL. While I've researched and structured this design carefully, it may not be perfect. If you're designing a similar HAL, consider conducting additional research and maybe do better design than this. But this chapter will show you the basics and how I did it.

## Prerequisites

In order to design the RTC HAL, we need to research and find what are the common functionalities of RTCs. We have to look at the datasheet of different RTC chips (in our case, I just referred to the DS1307 and DS3231). After going through them, I found these basic functions that RTCs support.

### Core RTC Functions

- Date Time Support: Obviously, this is the whole point of having an RTC. We should be able to set the date and time, and read back the current time whenever we need it. This is the most essential function in any RTC.
 
- Square Wave Support: If you've looked at the datasheets, you'll see they mention Square Wave functionality. This lets the RTC generate clock signals at various frequencies like 1Hz, 4kHz, 8kHz, or 32kHz. These signals are handy for system timing, triggering interrupts, or providing clock signals to other components.

- Power Control: Basic stuff like starting and stopping the clock.

### RTC Specific Features

Different RTC chips come with their own unique features:

- Non-volatile RAM: DS1307 has this feature which allows us to store 56 bytes of data that stays powered by the backup battery. This is useful for storing configuration data or small amount of information. The DS3231 doesn't have this.

- Alarm: The DS3231 has two independent alarm systems. You can configure them for different time and date combinations, and they can generate interrupts to wake up your system or trigger events. The DS1307 doesn't support alarms.

## What This Means for Our Design

We can't put all these functions into a single trait - we need it to be flexible. Even the core functions like square wave, I wasn't entirely sure if all RTCs support them. So I designed it in a way that splits up the traits.

We'll have a core trait called `Rtc` that handles the basic date and time functions. The RTC driver must implement this trait.

Then we'll have separate traits for different features: `RtcPowerControl`, `SquareWave`, and `RtcNvram`. These feature traits require the core Rtc trait, so any RTC that supports these features automatically has the basic functionality too.

For now, I'm not including the alarm trait in this design but will add it later (you can also try this as an exercise). The alarm system would take more sections to explain properly, so I want to keep things simple for now.

This isn't a perfect or final design. I actually tried different approaches with naming, error handling, and overall structure, and ended up modifying it multiple times. I finally settled on this design "for now".
