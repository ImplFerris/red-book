# Using Our DHT22 Crate with ESP32

Now that we've successfully built the driver for the DHT22 sensor and verified it using mock tests, it's time to test it on actual hardware. In this section, we'll connect the DHT22 sensor to the ESP32 and run a complete example. 

I won't be covering project setup or wiring basics for the ESP32 in this section. If you're new to embedded Rust or ESP32 development, I recommend checking out the book ["impl Rust for ESP32"](https://esp32.implrust.com).

## Circuit

| DHT22 Pin | Label on Module | Connects To (ESP32) | Description  |
| --------- | --------------- | ------------------- | ------------ |
| 1         | VCC (+)         | 3.3V                | Power Supply |
| 2         | DATA            | GPIO4               | Data Line    |
| 3         | GND (-)         | GND                 | Ground       |

> **Note:** DHT22 modules already include a 10kΩ pull-up resistor on the data line. If you're using a bare DHT22 sensor, you must add an external resistor (10kΩ) between VCC and DATA.

## Add the Crate as a Dependency

Create new project with esp-generate tool and add your Git repository as a dependency in your Cargo.toml:
```toml
[dependencies]
dht22-sensor = { git = "https://github.com/your-username/dht22-sensor" }
```

## Full Code
```rust
#![no_std]
#![no_main]
#![deny(
    clippy::mem_forget,
    reason = "mem::forget is generally not safe to do with esp_hal types, especially those \
    holding buffers for the duration of a data transfer."
)]

use defmt::info;
use dht22_sensor::{Dht22, DhtError};
use esp_hal::clock::CpuClock;
use esp_hal::delay::Delay;
use esp_hal::gpio::{self, Flex, Level};
use esp_hal::main;
use esp_println as _;

#[panic_handler]
fn panic(_: &core::panic::PanicInfo) -> ! {
    loop {}
}

esp_bootloader_esp_idf::esp_app_desc!();

#[main]
fn main() -> ! {
    // generator version: 0.4.0

    let config = esp_hal::Config::default().with_cpu_clock(CpuClock::max());
    let peripherals = esp_hal::init(config);

    let mut dht_pin = Flex::new(peripherals.GPIO4);

    let output_config = gpio::OutputConfig::default()
        .with_drive_mode(gpio::DriveMode::OpenDrain)
        .with_pull(gpio::Pull::None);
    dht_pin.apply_output_config(&output_config);
    dht_pin.set_input_enable(true);
    dht_pin.set_output_enable(true);
    dht_pin.set_level(Level::High);

    let mut delay = Delay::new();
    let delay1 = Delay::new();
    delay1.delay_millis(2000);

    let mut sensor = Dht22::new(&mut dht_pin, &mut delay);
    loop {
        match sensor.read() {
            Ok(reading) => {
                info!(
                    "Temperature: {:?}, Humidity: {:?}",
                    reading.temperature, reading.relative_humidity
                );
            }
            Err(err) => match err {
                DhtError::ChecksumMismatch => {
                    info!("checksum error");
                }
                DhtError::Timeout => {
                    info!("Timeout error");
                }
                DhtError::PinError(e) => {
                    info!("Pin error:{}", e);
                }
            },
        }
        delay1.delay_millis(5000);
    }

}
```

## Clone Existing Project

If you want to get started quickly, you can clone a ready-to-use example project from my repository:

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/non-async/dht-temphum
```

In that case, don't forget to remove the dht22-sensor crate from the Cargo.toml and add your Git repository instead.
