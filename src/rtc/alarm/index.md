{{#title Building an RTC Alarm Trait in Embedded Rust}}

# Adding RTC Alarm Support in Rust

We have already implemented the basic RTC functionality for date and time, square wave output, and NVRAM in the previous chapters. Now, we are going to extend the RTC HAL with another common feature: alarms.

The DS1307 does not have an alarm feature, but the DS3231 does. Many other RTCs also provide alarm functionality.

In this chapter, we will define a trait for RTC alarms and then implement it for the DS3231.

## Understanding RTC Alarms

An alarm is basically like a regular alarm we use. We set a date and time condition to match. When the RTC reaches that condition, it triggers the alarm.

An RTC can have one or more alarms. For example, the DS3231 has two independent alarms, Alarm 1 and Alarm 2. Each alarm can have its own configuration and can be enabled or disabled independently.

When an alarm condition is met, the RTC sets the corresponding alarm flag. The alarm can also be configured to generate an interrupt through the RTC's alarm output, allowing the MCU to respond to the event.

## Design Goal

The aim is to let the end user configure and enable an alarm with simple code:

```rust
let alarm = AlarmBuilder::at(7, 30).build()?;

rtc.set_alarm(AlarmId::Alarm1, alarm)?;
rtc.enable_alarm(AlarmId::Alarm1)?;
```

In this example, when the RTC reaches 07:30, Alarm 1 is triggered.
