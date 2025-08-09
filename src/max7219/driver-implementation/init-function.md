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

## Completed Project

I have made a simple driver project that follows everything we explained in this book. You can find it here: [https://github.com/ImplFerris/max7219-driver-project](https://github.com/ImplFerris/max7219-driver-project).

If you run into any errors while compiling or want to see a complete example, this project should be helpful.

## Where to Go From Here?

As i mentioned earlier, I have created a full-featured crate called **max7219-display** that provides easy-to-use utility functions to control both LED matrix and 7-segment displays. It is published on crates.io [here](https://crates.io/crates/max7219-display) 

We didn't cover this crate in detail because it's outside the scope of writing a low-level driver; it builds on top of the bare-driver we created in this book.

As an exercise, you can explore the crate's features and usage by checking out its documentation [here](https://docs.rs/max7219-display/latest/max7219_display/index.html)

You can also find the full source code on GitHub here:  
[https://github.com/ImplFerris/max7219-display](https://github.com/ImplFerris/max7219-display)
