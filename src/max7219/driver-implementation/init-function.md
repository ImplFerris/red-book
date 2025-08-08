# Init function

Now that we have defined all the necessary functions, we are almost done with the driver. The last function we need to implement is the `init` method.

After the user creates an instance of the `Max7219` driver, they must call this `init` function to perform the basic initialization of the MAX7219 chip. This sets up the device with a default state, ready for use.

The `init` function does the following:

- Powers on all devices in the chain  
- Disables the test mode on all devices  
- Sets the scan limit to cover all digits  
- Sets decode mode to NoDecode (raw data mode)
- Clears all display digits  

```rust
pub fn init(&mut self) -> Result<()> {
    self.power_on()?;

    self.test_all(false)?;
    self.set_scan_limit_all(NUM_DIGITS)?;
    self.set_decode_mode_all(DecodeMode::NoDecode)?;

    self.clear_all()?;

    Ok(())
}
```

Hoorah! We have finally completed our driver.

