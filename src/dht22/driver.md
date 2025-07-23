# Driver Implementation

We will now create the dht22 module and start implementing the driver logic inside it.

This module will contain the core functionality for communicating with the DHT22 sensor, including sending the start signal, reading bits, verifying the checksum, and returning the temperature and humidity data using the Reading struct.

To begin, add the following line in your lib.rs file to include the dht22 module:

```rust
pub mod dht22;

// re-export the Dht22 struct and Reading struct for library users
pub use dht22::{Dht22, Reading};
```
This makes the Dht22 driver and the Reading struct available to users of your library as your_crate::Dht22 and your_crate::Reading, instead of requiring them to manually access the submodule.

## Define the Reading Struct

Before we implement the driver logic, let's start with what we expect out of our module. The final goal of the driver is to return sensor readings.

We will define a simple Reading struct to hold the output values. It contains two fields: temperature (in Celsius) and relative humidity (as a percentage). This is what the user of our library will receive when they call the read method.

Create this inside your dht22.rs file:

```rust
#[cfg_attr(feature = "defmt", derive(defmt::Format))]
#[derive(Clone, Copy, Debug, PartialEq)]
pub struct Reading {
    /// Temperature in degrees Celsius.
    pub temperature: f32,
    /// Relative humidity in percent.
    pub relative_humidity: f32,
}
```

We use some useful traits like Clone, Copy, Debug, and PartialEq to make the Reading struct easy to work with. Along with these, we also derive defmt::Format, but only when the user enables the "defmt" feature. This allows the struct to be logged using the defmt crate. If the feature is not enabled, the derive is skipped automatically.

## Define the Dht22 Struct

The Dht22 struct is the core of our driver. It stores the pin used to communicate with the sensor and a delay object used to control timing. 

The DHT22 sensor uses a single-wire data line, which means the pin must switch between output mode (to send the start signal) and input mode (to read the response). That's why our "PIN" type must implement both the InputPin and OutputPin traits. Similarly, we need sub-millisecond delays to follow the timing requirements of the DHT22 protocol, so we use a delay object that implements the DelayNs trait.

Let's start by importing the required traits from embedded-hal:

```rust
use embedded_hal::{
    delay::DelayNs,
    digital::{InputPin, OutputPin},
};
```

Now define the struct:

```rust
pub struct Dht22<PIN, D> {
    pin: PIN,
    delay: D,
}
```

This struct is generic over the types "PIN" and "D". As mentioned, the pin must support both input and output modes, and the delay must support fine-grained timing.

Finally, we implement a simple constructor:

```rust
impl<PIN, DELAY, E> Dht22<PIN, DELAY>
where
    PIN: InputPin<Error = E> + OutputPin<Error = E>,
    DELAY: DelayNs,
{
     pub fn new(pin: PIN, delay: DELAY) -> Self {
        Dht22 { pin, delay }
    }

}
```


This allows the user to create a new instance of the driver by passing a pin and a delay object. The trait bounds ensure that the types they use can perform the required operations.

