# Project Setup

Now that we have enough information about the DS1307 device, let's start writing the code.

By the end of this chapter, our project structure will look like this:

```sh
├── Cargo.toml
├── src
│   ├── control.rs
│   ├── datetime.rs
│   ├── ds1307.rs
│   ├── error.rs
│   ├── lib.rs
│   ├── nvram.rs
│   ├── registers.rs
│   └── square_wave.rs
```

## Create new library

Let's initialize a new Rust library project for the DS1307 driver.

```sh
cargo new ds1307-rtc --lib
cd ds1307-rtc
```

## Update the Cargo.toml

### Features

We will configure optional features for our DS1307 driver to enable defmt logging capabilities when needed.

```toml

[features]
default = []
defmt = ["dep:defmt", "rtc-hal/defmt"]
```

### Dependencies

We need embedded-hal for the I2C traits. This lets us communicate with the DS1307 chip. rtc-hal is our RTC HAL that provides the traits our driver will implement. We also include defmt as an optional dependency. This is only enabled with the defmt feature. 

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

As usual, we need embedded-hal-mock for testing our DS1307 driver to simulate I2C communication without actual hardware.

```toml
[dev-dependencies]
embedded-hal-mock = { version = "0.11.1", "features" = ["eh1"] }
```
