# Initialization functions

In this section, we define functions that the end user will use to create and initialize new instances of the Max7219 struct. These functions provide the starting point for using the driver. Later, we will add other methods for the struct inside the same implementation block.

From here on, all functions should be placed inside this block:

```rust
impl<SPI> Max7219<SPI>
where
    SPI: SpiDevice,
{
    // Initialization and other methods go here
}
```

## Creating a new driver instance

This function creates a new Max7219 driver using the given SPI interface.  It defaults to controlling a single device. You can use with_device_count method later to specify more devices in a chain. 

> The SPI must use Mode 0 (clock low when idle, data read on the rising clock edge) and run at 10 MHz or less as required by the MAX7219 datasheet. We leave it to the user to make sure these settings are correct.

```rust
pub fn new(spi: SPI) -> Self {
    Self {
        spi,
        device_count: 1, // Default to 1, use with_device_count to increase count
        buffer: [0; MAX_DISPLAYS * 2],
    }
}
```

## Setting the number of devices

This method lets you set how many daisy-chained MAX7219 devices you want to control. If you try to set a number larger than the maximum allowed (MAX_DISPLAYS), it returns an error. For example, with LED matrices, you might have a single matrix, or 4 or 8 matrices chained together. This method lets you tell the driver how many are connected.

```rust
 pub fn with_device_count(mut self, count: usize) -> Result<Self> {
    if count > MAX_DISPLAYS {
        return Err(Error::InvalidDeviceCount);
    }
    self.device_count = count;
    Ok(self)
}
```

## Device Count Helper Function

We also create this function to let the user check how many MAX7219 devices are currently connected in the chain. It simply returns the device count stored inside the driver.

```rust
pub fn device_count(&self) -> usize {
    self.device_count
}
```

## Initialization from the User's Perspective

There is one more function we need to define called init() that will do basic setup for the MAX7219 and get it ready to use. I originally thought to include it in this section, but since it depends on other functions, we will add it later.

From the end user's perspective, they will create an instance of our driver and then initialize it like this:

```rust
// Example 
let driver = Max7219::new(spi_dev).with_device_count(4).unwrap();
driver.init().unwrap(); // we will define this later
```

## Tests

We will write simple tests to verify the basic behavior of the `new` and `with_device_count` functions. First, we check the default device count is 1. Next, we confirm setting a valid device count works correctly. Finally, we verify setting an invalid count returns the expected error. We will place these tests at the bottom of the max7219.rs module. Any other test functions we create later for the max7219 module should be placed inside the same mod tests block.

We use Spi Mock from embedded_hal_mock to simulate the SPI interface, ensuring our driver works correctly with the SpiDevice trait.

```rust

#[cfg(test)]
mod tests {
    use super::*;
    use crate::MAX_DISPLAYS;
    use embedded_hal_mock::eh1::spi::Mock as SpiMock;

    #[test]
    fn test_new() {
        let mut spi = SpiMock::new(&[]);
        let driver = Max7219::new(&mut spi);
        // Default device count => 1
        assert_eq!(driver.device_count(), 1);

        spi.done();
    }

    #[test]
    fn test_with_device_count_valid() {
        let mut spi = SpiMock::new(&[]);
        let driver = Max7219::new(&mut spi);
        let driver = driver
            .with_device_count(4)
            .expect("Should accept valid count");
        assert_eq!(driver.device_count(), 4);
        spi.done();
    }

    #[test]
    fn test_with_device_count_invalid() {
        let mut spi = SpiMock::new(&[]);
        let driver = Max7219::new(&mut spi);
        let result = driver.with_device_count(MAX_DISPLAYS + 1);
        assert!(matches!(result, Err(Error::InvalidDeviceCount)));

        spi.done();
    }
}
```
  