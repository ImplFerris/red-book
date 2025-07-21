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
#![no_std]
```

This tells the Rust compiler that our crate should not link against the standard library. As a result, we won't be able to use any features from the std crate - only what's available in core.

## Add Dependencies

Next, open Cargo.toml and add the embedded-hal crate under [dependencies]. This allows your driver to work with any compatible GPIO or delay implementation:

```
[dependencies]
embedded-hal = "1.0.0"
```
