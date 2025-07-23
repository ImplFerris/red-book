# Create Project

Now that we have a solid understanding of the DHT22 sensor's communication protocol and data format, let's begin implementing the driver in Rust for a no_std environment.

We'll start by creating a new Rust library crate. This crate will be minimal and portable, relying only on embedded-hal traits so it can run on any platform that implements them.

## Create new library

```sh
cargo new dht22-sensor --lib
cd dht22-sensor
```

This creates a basic Rust library named dht22-sensor with a src/lib.rs entry point.

## Mark the Crate as no_std
To make our library compatible with embedded systems that don't have access to the standard library, we need to mark it as no_std.

Open the src/lib.rs file, remove all lines and add the following line at the very top:

```rust
#![cfg_attr(not(test), no_std)]
```

This tells the Rust compiler to avoid linking the standard library when we are not running tests. Instead, the crate will rely only on the core crate, which provides essential Rust features like basic types and operations. As a result, we won't be able to use any features from the std crate - only what's available in core.

The cfg_attr is a conditional attribute. It means "if not testing, then apply no_std." This way, we can still write normal unit tests using std, but the compiled library will work in no_std environments.

## Add Dependencies

Next, open Cargo.toml and add the embedded-hal crate under [dependencies]. This allows your driver to work with any compatible GPIO or delay implementation:

```toml
[dependencies]
embedded-hal = "1.0.0"
```

### Optional: defmt Logging Support

We also add optional defmt support so that our driver can integrate with the defmt logging ecosystem.

To enable this, users must explicitly activate the defmt feature.

Add the following to your Cargo.toml:

## Defmt
```toml
defmt = { version = "1.0.1", optional = true }

[features]
defmt = ["dep:defmt"]
```

## Goal

By the end of this tutorial, we will have the following file structure for our DHT22 driver. While this is a relatively simple driver and could be implemented entirely in lib.rs, we prefer a more modular structure for clarity and maintainability.

> The lib.rs file is the main entry point of a Rust library crate. When someone adds your crate as a dependency and uses use your_crate::..., the items they access come from what you expose in lib.rs. In our case, lib.rs will include and publicly re-export the dht22 module and the error module, making them accessible to users of the crate. Think of it as the public API surface of your driver.

```sh
.
├── Cargo.toml
└── src
    ├── dht22.rs
    ├── error.rs
    └── lib.rs
```


## Completed Project

If you ever get stuck or feel unsure about how everything fits together, you can refer to the complete working project I have created for this chapter. You'll find it on GitHub at [implferris/dht22-sensor](https://github.com/implferris/dht22-sensor). It contains all the source code, tests, and structure exactly as discussed here. Feel free to explore it and compare with your own implementation.
