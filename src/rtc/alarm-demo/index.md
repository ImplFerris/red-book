{{#title How to Manage DS3231 RTC Alarms on ESP32 with Embedded Rust | RED Book}}

# How to Manage DS3231 RTC Alarms on ESP32 with Embedded Rust

Now that we have implemented the `Alarm` trait for the DS3231 driver, let's create a demo application to test it.

We will use an ESP32 with a DS3231 RTC. The DS3231 `INT/SQW` pin will be connected to GPIO4 of the ESP32. When the alarm triggers, the DS3231 will pull this pin low. The ESP32 will detect the falling edge using a GPIO interrupt.

In this example, we will configure the alarm to trigger when the seconds value reaches `30` (for example, at `10:05:30`, `10:06:30`, and so on). It does not mean that the alarm triggers every 30 seconds. When the alarm is triggered, the ESP32 will toggle the onboard LED and print a message.

## Hardware Connections

Connect the DS3231 to the ESP32 as follows:

| DS3231 | ESP32 |
| --- | --- |
| `SCL` | `GPIO22` |
| `SDA` | `GPIO21` |
| `INT/SQW` | `GPIO4` |
| `VCC` | `3.3V` |
| `GND` | `GND` |

## Project Setup

You can use `esp-generate` to create the project and build on top of it. I will not go into the details of creating the project, as I assume you are already familiar with the basics.

If you need help setting up an ESP32 Rust project, you can also refer to my [ESP32 with Rust book](https://esp32.implrust.com/).

> [!NOTE]
> The complete example is available in the [`esp32/alarm`](https://github.com/ImplFerris/ds3231-examples/tree/main/esp32/alarm) folder of the [`ds3231-examples`](https://github.com/ImplFerris/ds3231-examples) repository.


Add `embedded-hal-bus` and the DS3231 driver to `Cargo.toml`:

```toml
embedded-hal-bus = "0.3.0"

# ds3231-rtc = { version = "0.3.0", optional = true }

ds3231-rtc = { git = "https://github.com/<YOUR_USERNAME>/ds3231-rtc", optional = true }
```

You can use your own version of the ds3231-rtc repository, or use mine.

## Handling the GPIO Interrupt

As you saw earlier, we connected the DS3231 `INT/SQW` pin to GPIO4 of the ESP32. When the alarm is triggered, the DS3231 pulls this pin low. We need to handle this interrupt in our ESP32 code and let the main loop know that the alarm occurred. For this, we will set the `ALARM` variable from the interrupt handler.

```rust
static ALARM: AtomicBool = AtomicBool::new(false);
```

To handle the interrupt, we need to define an interrupt handler. The ESP32 GPIO interrupt handler is generic, so it is not associated with a specific GPIO pin. Inside the handler, we need to check whether GPIO4 caused the interrupt and clear the interrupt flag for the pin.

For this, the interrupt handler needs access to the GPIO4 input pin. We cannot simply pass the pin as an argument to the interrupt handler because the ESP32 interrupt handler is called by the hardware and has a fixed function signature.

We will therefore make the input pin accessible globally. We use `Mutex` from the `critical_section` crate together with `RefCell` to safely access the pin from the interrupt handler.

```rust
static ALARM_PIN: Mutex<RefCell<Option<Input>>> = Mutex::new(RefCell::new(None));

#[handler]
fn handler() {
    critical_section::with(|cs| {
        info!("GPIO interrupt");
        if let Some(pin) = ALARM_PIN.borrow_ref_mut(cs).as_mut()
            && pin.is_interrupt_set()
        {
            pin.clear_interrupt();
            ALARM.store(true, Ordering::Relaxed);
        }
    });
}
```

The handler first checks whether the alarm pin caused the interrupt. If it did, we clear the GPIO interrupt and set the `ALARM` flag.

In the `main` function, we register the interrupt handler for GPIO interrupts:

```rust
let mut io = Io::new(peripherals.IO_MUX);
io.set_interrupt_handler(handler); // Set the interrupt handler for GPIO interrupts.
```

## Configuring the Alarm Pin

We now configure GPIO4 as an input with an internal pull-up and listen for a falling-edge interrupt. We then store the pin in the global `ALARM_PIN` so the interrupt handler can access it.

```rust
let mut alarm_pin = Input::new(
    peripherals.GPIO4,
    InputConfig::default().with_pull(Pull::Up),
);

critical_section::with(|cs| {
    alarm_pin.listen(Event::FallingEdge);
    ALARM_PIN.borrow_ref_mut(cs).replace(alarm_pin);
});
```

## LED

We will use the onboard LED of the ESP32 DevKit V1 to indicate when the alarm interrupt occurs.

```rust
let mut led = Output::new(peripherals.GPIO2, Level::High, OutputConfig::default());
```

## Configuring the Alarm

We will configure Alarm 1 to trigger when the seconds value reaches `30`. We first create the alarm using `AlarmBuilder`, set it on the DS3231, clear any existing alarm flag, and then enable the alarm.

```rust
let alarm = AlarmBuilder::at_second(30).build().unwrap();
rtc.set_alarm(AlarmId::Alarm1, alarm).unwrap();
rtc.clear_alarm(AlarmId::Alarm1).unwrap();
rtc.enable_alarm(AlarmId::Alarm1).unwrap();
```

## Handling the Alarm

The interrupt handler sets the `ALARM` flag when the alarm interrupt occurs. In the main loop, we check this flag and take action when it is set.

We toggle the LED, clear the DS3231 alarm flag, and print a message.

```rust
loop {
    if ALARM.swap(false, Ordering::Relaxed) {
        led.toggle();

        rtc.clear_alarm(AlarmId::Alarm1).unwrap();

        info!("Alarm triggered!");
    }
    let delay_start = Instant::now();
    while delay_start.elapsed() < Duration::from_secs(1) {}
}
```

This is just a simple example. In a real application, the application can continue performing other tasks while waiting for the alarm.

## The Full Code

```rust
#![no_std]
#![no_main]
#![deny(
    clippy::mem_forget,
    reason = "mem::forget is generally not safe to do with esp_hal types, especially those \
    holding buffers for the duration of a data transfer."
)]

use core::cell::RefCell;
use core::sync::atomic::{AtomicBool, Ordering};

use critical_section::Mutex;
use defmt::info;
use ds3231_rtc::alarm::AlarmId;
use esp_hal::clock::CpuClock;
use esp_hal::gpio::{Event, Input, InputConfig, Io, Level, Output, OutputConfig, Pull};
use esp_hal::time::{Duration, Instant};
use esp_hal::{handler, main};
use esp_println as _;

use ds3231_rtc::Ds3231;
use ds3231_rtc::{Alarm, AlarmBuilder, Rtc};
use esp_hal::time::Rate;

#[panic_handler]
fn panic(err: &core::panic::PanicInfo) -> ! {
    defmt::error!("Error:{:?}", err);
    loop {}
}

// This creates a default app-descriptor required by the esp-idf bootloader.
// For more information see: <https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/app_image_format.html#application-description>
esp_bootloader_esp_idf::esp_app_desc!();

static ALARM: AtomicBool = AtomicBool::new(false);
static ALARM_PIN: Mutex<RefCell<Option<Input>>> = Mutex::new(RefCell::new(None));

#[handler]
fn handler() {
    critical_section::with(|cs| {
        info!("GPIO interrupt");
        if let Some(pin) = ALARM_PIN.borrow_ref_mut(cs).as_mut()
            && pin.is_interrupt_set()
        {
            pin.clear_interrupt();
            ALARM.store(true, Ordering::Relaxed);
        }
    });
}

#[main]
fn main() -> ! {
    // generator version: 0.4.0

    let config = esp_hal::Config::default().with_cpu_clock(CpuClock::max());
    let peripherals = esp_hal::init(config);

    let i2c_bus = esp_hal::i2c::master::I2c::new(
        peripherals.I2C0,
        esp_hal::i2c::master::Config::default().with_frequency(Rate::from_khz(400)),
    )
    .unwrap()
    .with_scl(peripherals.GPIO22)
    .with_sda(peripherals.GPIO21);

    let mut rtc = Ds3231::new(i2c_bus);
    // let dt = DateTime::new(2025, 8, 21, 20, 11, 30).unwrap();
    // rtc.set_datetime(&dt).unwrap();

    let dt = rtc.get_datetime().expect("Unable to get date time");
    info!("Year: {}", dt.year());
    info!("Month: {}", dt.month());
    info!("Day Of month: {}", dt.day_of_month());
    info!("Hour: {}", dt.hour());
    info!("Minute: {}", dt.minute());
    info!("Seconds: {}", dt.second());
    info!("Day of week: {}", dt.calculate_weekday().unwrap().as_str());

    let mut led = Output::new(peripherals.GPIO2, Level::High, OutputConfig::default());

    let mut io = Io::new(peripherals.IO_MUX);
    io.set_interrupt_handler(handler); // Set the interrupt handler for GPIO interrupts.

    let mut alarm_pin = Input::new(
        peripherals.GPIO4,
        InputConfig::default().with_pull(Pull::Up),
    );

    critical_section::with(|cs| {
        alarm_pin.listen(Event::FallingEdge);
        ALARM_PIN.borrow_ref_mut(cs).replace(alarm_pin);
    });

    let alarm = AlarmBuilder::at_second(30).build().unwrap();

    rtc.set_alarm(AlarmId::Alarm1, alarm).unwrap();
    rtc.clear_alarm(AlarmId::Alarm1).unwrap();
    rtc.enable_alarm(AlarmId::Alarm1).unwrap();

    loop {
        if ALARM.swap(false, Ordering::Relaxed) {
            led.toggle();

            rtc.clear_alarm(AlarmId::Alarm1).unwrap();

            info!("Alarm triggered!");
        }
        let delay_start = Instant::now();
        while delay_start.elapsed() < Duration::from_secs(1) {}
    }

    // for inspiration have a look at the examples at https://github.com/esp-rs/esp-hal/tree/esp-hal-v1.0.0-beta.1/examples/src/bin
}
```
