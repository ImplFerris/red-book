# Demo App

Now we have the RTC HAL crate and two drivers (DS1307 and DS3231) that implement the traits provided by the RTC HAL. Next, we will make a demo app to show how this works. I will make a project for the ESP32 chip.

In this project, we will have two feature flags: "ds1307" (turned on by default) and "ds3231" for the demo project. I will not explain all the details about making this project (you should already know how to do this) or how to set up these feature flags. I will only talk about the important parts. I recommend you look at this repository for the complete project: "[https://github.com/implferris/rtc-hal-demo](https://github.com/implferris/rtc-hal-demo)"

## Cargo.toml file

### Feature flag
The feature flags define which RTC driver to include in the build. We set DS1307 as the default.

```toml
[features]
default = ["ds1307"]        # DS1307 is the default
ds3231 = ["dep:ds3231-rtc"]
ds1307 = ["dep:ds1307-rtc"]
```

### Additional Dependency

We mark RTC driver dependency as optional and will only be included when their corresponding feature is enabled.

> Here: replace rtc-hal, ds3231-rtc, and ds1307-rtc with your GitHub url or local folder path.

```toml
# ...
embedded-hal-bus = "0.3.0"

# rtc-hal = { version = "0.3.0", features = ["defmt"] }
# ds3231-rtc = { version = "0.2.2", optional = true }
# ds1307-rtc = { version = "0.2.2", optional = true }

rtc-hal = { git = "https://github.com/<YOUR_USERNAME>/rtc-hal", features = ["defmt"] }
ds3231-rtc = { git = "https://github.com/<YOUR_USERNAME>/ds3231-rtc", optional = true }
ds1307-rtc = { git = "https://github.com/<YOUR_USERNAME>/ds1307-rtc", optional = true }
```


## App module (app.rs)

This is the main app code that works with any RTC driver.

```rust
use esp_alloc as _;
use rtc_hal::error::{Error, ErrorKind};
use rtc_hal::square_wave::{SquareWave, SquareWaveFreq};
use rtc_hal::{datetime::DateTime, rtc::Rtc};

type Result<T> = core::result::Result<T, ErrorKind>;

pub struct DemoApp<RTC> {
    rtc: RTC,
}

impl<RTC> DemoApp<RTC>
where
    RTC: Rtc,
{
    pub fn new(rtc: RTC) -> Self {
        Self { rtc }
    }

    pub fn set_datetime(&mut self, dt: &DateTime) -> Result<()> {
        self.rtc.set_datetime(dt).map_err(|e| e.kind())?;
        Ok(())
    }

    pub fn print_current_time(&mut self) -> Result<()> {
        let current_time = self.rtc.get_datetime().map_err(|e| e.kind())?;

        defmt::info!(
            "📅 {}-{:02}-{:02} 🕐 {:02}:{:02}:{:02}",
            current_time.year(),
            current_time.month(),
            current_time.day_of_month(),
            current_time.hour(),
            current_time.minute(),
            current_time.second()
        );
        Ok(())
    }
}

impl<RTC> DemoApp<RTC>
where
    RTC: SquareWave,
{
    pub fn start_square_wave(&mut self) -> Result<()> {
        self.rtc
            .start_square_wave(SquareWaveFreq::Hz1)
            .map_err(|e| e.kind())?;
        Ok(())
    }

    pub fn stop_square_wave(&mut self) -> Result<()> {
        self.rtc.disable_square_wave().map_err(|e| e.kind())?;
        Ok(())
    }
}
```

Let's break it down.

### Main struct

The app uses generics (<RTC>) so it can work with any RTC chip that implements our RTC HAL traits. This means the same code works for both DS1307 and DS3231.

```rust
pub struct DemoApp<RTC> {
    rtc: RTC,
}
```

### Basic RTC Functions:

The first impl block needs the RTC type to have the Rtc trait. It lets you set and get current datetime in a nice format.

```rust
impl<RTC> DemoApp<RTC>
where
    RTC: Rtc,
{
...
}
```

### Square Wave Functions:

The second impl block needs the RTC type to have the SquareWave trait. In the demo, we let users start or stop the square wave output.

```rust
impl<RTC> DemoApp<RTC>
where
    RTC: SquareWave,
{
...
}
```

## Main module (main.rs)

In the main module, we first set up the i2c bus. I have connected the SCL pin of RTC to GPIO pin 22 of ESP32 Devkit v1 and SDA pin of RTC to GPIO pin 21 of ESP32 Devkit v1.

```rust
let i2c_bus = esp_hal::i2c::master::I2c::new(
        peripherals.I2C0,
        esp_hal::i2c::master::Config::default().with_frequency(Rate::from_khz(100)),
    )
    .unwrap()
    .with_scl(peripherals.GPIO22)
    .with_sda(peripherals.GPIO21);
```
Note: We use 100 kHz speed which works well with both RTC chips.


### Create the Initial Date and Time

We create a datetime object that we'll use to set the initial time on the RTC chip. This is like setting a clock when you first get it.

```rust
let dt = DateTime::new(2025, 9, 2, 23, 41, 30).unwrap();
```
Note: This creates a datetime object for September 2, 2025 at 23:41:30 (11:41:30 PM). You can change these values to match the current date and time when you run the program.

### Choose Which RTC Driver to Use

Here's where our feature flags come into play. We let the compiler pick which RTC driver to use based on what features are turned on.

```rust
#[cfg(feature = "ds3231")]
let rtc = ds3231_rtc::Ds3231::new(i2c_bus);

#[cfg(not(feature = "ds3231"))]
let rtc = ds1307_rtc::Ds1307::new(i2c_bus);
```

## Create Our Demo App

We create our demo app with whichever RTC driver was picked above. This shows the real power of our RTC HAL design.

```rust
let mut app = DemoApp::new(rtc);
```
The same app code works with different chips without any changes. Our app doesn't need to know if it's talking to a DS1307 or DS3231 - it just uses the common traits we defined.

## Run the RTC Demo

Finally, we run through a complete demo that shows all the RTC features working together.

```rust
info!("setting datetime");
app.set_datetime(&dt).unwrap();

info!("starting square wave");
app.start_square_wave().unwrap();

let delay_start = Instant::now();
while delay_start.elapsed() < Duration::from_secs(60) {}
info!("stopping square wave");
app.stop_square_wave().unwrap();

loop {
    info!("----------");
    info!("getting datetime");
    info!("----------");
    if let Err(e) = app.print_current_time() {
        info!("RTC Error: {}", e);
    }
    let delay_start = Instant::now();
    while delay_start.elapsed() < Duration::from_minutes(1) {}
}
```
