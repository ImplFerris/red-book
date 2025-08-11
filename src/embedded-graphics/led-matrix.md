# LED Matrix

We will not implement embedded-graphics directly for the Max7219 struct because we want to keep it focused on core functions for both 7-segment and LED matrix control. Since embedded-graphics is not very useful for 7-segment displays, we'll create a separate struct called LedMatrix dedicated to LED matrix displays.

We will follow the [minimum implementation](https://docs.rs/embedded-graphics-core/latest/embedded_graphics_core/draw_target/trait.DrawTarget.html#minimum-implementation) example from the embedded-graphics-core crate. That example shows a fake display with a fixed size of 64x64 pixels. It defines a framebuffer as framebuffer: [u8; 64 * 64], with one u8 per pixel, so the buffer length is 4096 bytes. The example struct also includes an interface (iface) like SPI to send data to the display.

In our case, for the interface part, we will provide the Max7219 driver object that takes care of the SPI communication.

## Framebuffer length

We can't fix the framebuffer length like that. For a single 8x8 LED matrix, the buffer length would be 64 bytes (8 * 8), but we want to support daisy-chained devices. The user might have a single LED matrix or 4 or 8 daisy-chained matrices.

- For 4 daisy-chained matrices, the display size is 32x8 pixels, requiring a buffer length of 256.

- For 8 daisy-chained matrices, it's 64x8 pixels, requiring a buffer length of 512.

> NOTE: In the LED matrix, each pixel corresponds directly to one physical LED, represented by 1 byte per pixel in the framebuffer (for on/off).

So the framebuffer length must be dynamic depending on the number of chained devices. To solve this, we have different approaches:

1. **Use a fixed-size buffer with a maximum supported size.**  
   We can define a buffer large enough to handle the maximum number of chained matrices we expect to support (for example, 8 matrices -> 512 bytes). The buffer will always be this size, but we only use the part needed based on how many devices are connected.  
   The downside is that we over-allocate memory for setups with fewer matrices (like just 1 or 4 devices).

2. **Use heapless data structures.**  
   The [`heapless`](https://docs.rs/heapless/latest/heapless/) crate offers fixed-capacity containers like `Vec` and `ArrayVec` that work without dynamic allocation. This allows dynamic-length buffers within a fixed capacity.  
   However, this adds an external dependency, and I want to keep dependencies minimal; especially for a library.  
   (In fact, in my original `max7219-display` crate, I even put `embedded-graphics` behind a feature flag to keep it minimal and optional.)

3. **Use a generic const parameter for buffer size.**  
   We can make the `LedMatrix` struct generic over a const parameter representing the number of chained devices and total pixels. This lets us define the buffer size at compile time, avoiding dynamic allocation but still supporting different sizes.
 
So our final struct we will define it like this (in src/led_matrix/display.rs file)

```rust
pub struct LedMatrix<SPI, const BUFFER_LENGTH: usize = 64, const DEVICE_COUNT: usize = 1> {
    driver: Max7219<SPI>,
    /// Each 8x8 display has 64 pixels. For `N` daisy-chained devices,
    /// the total framebuffer size is `N * 64` pixels.
    ///
    /// For example, with 4 devices: `4 * 64 = 256` pixels.
    ///
    framebuffer: [u8; BUFFER_LENGTH],
}
```

## Re-export the LedMatrix

Optionally, you can update the led_matrix/mod.rs file to re-export the LedMatrix struct. This makes it easier for end users to access it. Instead of using `crate_name::led_matrix::display::LedMatrix`, they can simply write `crate_name::led_matrix::LedMatrix`

To do this, add the following to `led_matrix/mod.rs` file:

```rust
mod display;

pub use display::LedMatrix;
```

## Initialization

Let's get back to the display.rs module and implement some basic functions that will help initialize the LedMatrix struct.

```rust

impl<SPI, const BUFFER_LENGTH: usize, const DEVICE_COUNT: usize>
    LedMatrix<SPI, BUFFER_LENGTH, DEVICE_COUNT>
where
    SPI: SpiDevice,
{
    pub fn from_driver(driver: Max7219<SPI>) -> Result<Self> {
        if driver.device_count() != DEVICE_COUNT {
            return Err(Error::InvalidDeviceCount);
        }
        Ok(Self {
            driver,
            framebuffer: [0; BUFFER_LENGTH],
        })
    }

    pub fn driver(&mut self) -> &mut Max7219<SPI> {
        &mut self.driver
    }
}
```

We defined the from_driver function that accepts an already initialized Max7219 driver object. It verifies that the driver's device count matches the LedMatrix's expected device count, otherwise it returns an error. The framebuffer is initialized with zeros, sized by the BUFFER_LENGTH generic const parameter.

> In my original max7219-display crate, I also implemented another convenience function from_spi to simplify initialization. It accepts a SpiDevice and creates the driver internally in one step.

The driver() method is straightforward: it returns a mutable reference to the driver. This lets users access and manipulate driver-level functions directly if needed.

## Example initialization

For 4 daisy-chained devices:

```rust
let driver = Max7219::new(&mut spi_dev)
    .with_device_count(4)
    .expect("4 is valid count");

let display: LedMatrix<_, 256,4> = LedMatrix::from_driver(driver).expect("valid initialzation of the display");
```

For a single device, using default device count and buffer length:

```rust
let driver = Max7219::new(&mut spi_dev);    
let display: LedMatrix<_> = LedMatrix::from_driver(driver).expect("valid initialization of the default display");
```


## Tests


```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::registers::Register;
    use embedded_hal_mock::eh1::spi::Mock as SpiMock;
    use embedded_hal_mock::eh1::spi::Transaction;
    
    #[test]
    fn test_from_driver() {
        let mut spi = SpiMock::new(&[]);
        let driver = Max7219::new(&mut spi);
        let matrix: LedMatrix<_, 64, 1> = LedMatrix::from_driver(driver).unwrap();
        assert_eq!(matrix.framebuffer, [0u8; 64]);
        spi.done();
    }

    #[test]
    fn test_driver_mut_access() {
        let expected_transactions = [
            Transaction::transaction_start(),
            Transaction::write_vec(vec![Register::Shutdown.addr(), 0x01]),
            Transaction::transaction_end(),
        ];
        let mut spi = SpiMock::new(&expected_transactions);
        let original_driver = Max7219::new(&mut spi);
        let mut matrix: LedMatrix<_> = LedMatrix::from_driver(original_driver).unwrap();

        let driver = matrix.driver();

        driver.power_on().expect("Power on should succeed");
        spi.done();
    }

}
```
