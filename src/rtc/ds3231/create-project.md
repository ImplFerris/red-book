# Project Setup

Let's our implementation for the DS3231 driver also. By the end of this chapter, ds3231 project structure will look like this:

```sh
├── Cargo.toml
├── src
│   ├── control.rs
│   ├── datetime.rs
│   ├── ds3231.rs
│   ├── error.rs
│   ├── lib.rs
│   ├── registers.rs
│   └── square_wave.rs
```

The project structure (and the code) is almost the same as the DS1307 Driver. If you have noticed, we don't have the nvram module for ds3231 since it does not have one, so we won't be implementing that.

> ⚡ Challenge: I recommend trying to write the DS3231 Driver yourself. This is the best way to learn it deeply. It should be easy since you already saw how we did it for the DS1307. You can come back and check this chapter later.

I won't explain some of the code in detail since it was already covered in the DS1307 chapter.

## Create new library

Let's initialize a new Rust library project for the DS3231 driver.

```sh
cargo new ds3231-rtc --lib
cd ds3231-rtc
```

## Update the Cargo.toml

### Dependencies

We need embedded-hal for the I2C traits. This lets us communicate with the DS3231 chip. rtc-hal is our RTC HAL that provides the traits our driver will implement. We also include defmt as an optional dependency. This is only enabled with the defmt feature.

```toml
[dependencies]
embedded-hal = "1.0.0"
# rtc-hal = { version = "0.3.0", default-features = false }
rtc-hal = { git = "https://github.com/<YOUR_USERNAME>/rtc-hal", default-features = false }
defmt = { version = "1.0.1", optional = true }
```

I have published rtc-hal to crates.io. In your case, you don't need to publish it to crates.io. You can publish it to GitHub and use it.

Or you can use the local path:

```toml
rtc-hal = { path = "../rtc-hal", default-features = false }
```

### Dev Dependencies

As usual, we need embedded-hal-mock for testing our DS3231 driver to simulate I2C communication without actual hardware.

```toml
[dev-dependencies]
embedded-hal-mock = { version = "0.11.1", "features" = ["eh1"] }
```

### Features

We will configure optional features for our DS3231 driver to enable defmt logging capabilities when needed.

```toml
[features]
default = []
defmt = ["dep:defmt", "rtc-hal/defmt"]
```
