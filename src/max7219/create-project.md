# Create Project

Now that we understand the MAX7219 chip and how data is sent to control the display, we can start building our driver in Rust for a `no_std` environment.

We will begin by creating a new Rust library crate. The crate will be kept minimal and portable, using only the `embedded-hal` traits so it can work on any platform that supports them.

## Create new library

I named my crate `max7219-display`, but you can choose any name you like.

```rust
cargo new max7219-display --lib
cd max7219-display
```

## Goal

Our goal is to build a minimal driver that can send data to the MAX7219 and provide low-level functions to control the display.  

The simplest version could live entirely in `lib.rs`, but I prefer keeping things organized in separate modules. This makes the code cleaner and easier to scale later.

A final project layout will look like this:

```sh
.
├── Cargo.toml
└── src
    ├── driver
    │   ├── max7219.rs
    │   └── mod.rs
    ├── error.rs
    ├── lib.rs
    ├── registers.rs
```

I've already created a full-featured `max7219-display` crate and published it on [crates.io](https://crates.io/crates/max7219-display), with the source code available on [GitHub](https://github.com/ImplFerris/max7219-display). That version includes extra utilities for working with both LED matrices (single and daisy-chained) and 7-segment displays.

In this guide, we'll focus only on the bare minimum driver. This will give you the core understanding, and if you want, you can later extend it with your own utility functions. I won't cover those extras here so the guide stays focused and uncluttered.

For reference, here's the structure of my full crate:

```sh
.
├── Cargo.toml
└── src
    ├── driver
    │   ├── max7219.rs
    │   └── mod.rs
    ├── error.rs
    ├── led_matrix
    │   ├── buffer.rs
    │   ├── display.rs
    │   ├── fonts.rs
    │   ├── mod.rs
    │   ├── scroll.rs
    │   └── symbols.rs
    ├── lib.rs
    ├── registers.rs
    └── seven_segment
        ├── display.rs
        ├── fonts.rs
        └── mod.rs
```

## Creating the Module Structure for Our Driver

We will set up the basic folder and file structure.   We are only setting up the structure at this stage. We will add the actual code in the later sections when we start implementing our driver functionality.

Inside the src folder, create the following:

```sh
driver/max7219.rs
driver/mod.rs
registers.rs
error.rs
```

Add the following to `driver/mod.rs` file:

```rust
mod max7219; 
```

Here we have declared the max7219 module as private, meaning it can only be accessed within the driver module. 

You might wonder - if it is private, how will we use it outside?

Later, we will create a main struct for our driver called "Max7219" and export only that struct publicly. This approach keeps our internal code hidden and presents a clean public API.

You do not need to strictly follow this approach. If you prefer, you could also make the module public right away by writing `pub mod max7219` instead.

In Rust, the lib.rs file is the entry point for a library.
 
We will declare our modules here:

```rust
pub mod driver;
pub mod error;
pub mod registers;
```

That is all folks! The driver is done! Okay, not really - we just created the structure. We will add the actual code in the next section.
