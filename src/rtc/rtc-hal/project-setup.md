# Project Setup

This is what we are aiming for:
```sh
.
├── Cargo.toml
├── README.md
└── src
    ├── alarm.rs
    ├── bcd.rs
    ├── control.rs
    ├── datetime.rs
    ├── error.rs
    ├── lib.rs
    ├── nvram.rs
    ├── rtc.rs
    └── square_wave.rs
```

Each module corresponds to different parts of our RTC HAL design. The rtc.rs contains our core trait, while square_wave.rs, nvram.rs, and control.rs handle the feature-specific traits. The datetime.rs module deals with date and time types, bcd.rs provides utillity functions for Binary Coded Decimal conversions (since most RTCs work with BCD internally), and error.rs defines our error types. Even though we're not covering alarms in this section, I've included alarm.rs for future implementation (it is just with comment "//TODO").

## Create new library

Let's initialize a new Rust library project.

```sh
cargo new rtc-hal --lib
cd rtc-hal
```

Feel free to rename the project name. You can create all the necessary files and modules with placeholder content upfront, or create them as we go along.

## Finally the main module(lib.rs) will have this

```rust
pub mod alarm;
pub mod bcd;
pub mod control;
pub mod datetime;
pub mod error;
pub mod nvram;
pub mod rtc;
pub mod square_wave;
```

## Configuring Dependencies

For this library, we'll include defmt as an optional dependency to provide structured logging capabilities. We will put this dependency behind feature flag, so users can choose whether to include defmt support in their builds, keeping the library lightweight for those who don't need it.

Update your Cargo.toml with the following:

```toml
[dependencies]
defmt = { version = "1.0.1", optional = true }

[features]
defmt = ["dep:defmt"]
```

