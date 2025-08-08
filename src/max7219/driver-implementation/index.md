# Write an Embedded Rust Driver for the MAX7219

Now, we will start working on the core functionalities of the MAX7219 driver. We will create a struct called Max7219. This struct will be exposed to the user so they can use it to control the MAX7219 chip.

Before getting into that, let's declare two constants in the main lib module (lib.rs file). We need these for our Max7219 struct:

```rust
/// Maximum number of daisy-chained displays supported
pub const MAX_DISPLAYS: usize = 8;

/// Number of digits (0 to 7) controlled by one MAX7219
pub const NUM_DIGITS: u8 = 8;
```

After this, we re-export the Max7219 struct so users can access it:

Note: At this point, you will get a compiler error because we haven't defined the Max7219 struct yet. Don't worry, we will create it in the next steps.

```rust
pub use max7219::Max7219;
```

From now on, we will be working on the max7219 module (driver/max7219.rs file).

## Imports

To start, we import the SpiDevice trait from embedded-hal since our driver will communicate over SPI. We also bring in constants like MAX_DISPLAYS and NUM_DIGITS, our custom Result type, the Error enum for error handling, and the DecodeMode and Register enums from the registers module. 

```rust
use embedded_hal::spi::SpiDevice;

use crate::{
    MAX_DISPLAYS, NUM_DIGITS, Result,
    error::Error,
    registers::{DecodeMode, Register},
};
```

## Max7219 struct

Let's start with defining the main struct. This struct will hold the generic SPI interface, a buffer for preparing data, and the count of devices connected in a daisy chain.

```rust
/// Driver for the MAX7219 LED display controller.
/// Communicates over SPI using the embedded-hal `SpiDevice` trait.
pub struct Max7219<SPI> {
    spi: SPI,
    buffer: [u8; MAX_DISPLAYS * 2],
    device_count: usize,
}
```

The struct is generic over the SPI type because we want it to work with any SPI device that implements the embedded-hal SpiDevice trait. This makes our driver flexible and compatible with many hardware platforms.

The buffer is a fixed-size array used to build the full SPI data packet for all connected devices. Each MAX7219 device expects a 16-bit packet: 1 byte for the register address and 1 byte for the data. When multiple devices are chained together, we send data for all devices. The buffer size is MAX_DISPLAYS * 2 because we reserve space for the maximum number of devices.

The device_count keeps track of how many MAX7219 devices are connected in the chain. This is important because we only fill and send data for the connected devices, not the full maximum buffer.
