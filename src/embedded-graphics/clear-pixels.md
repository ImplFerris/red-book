# Clear Pixels

We have created functions to draw on the framebuffer, then push the framebuffer to the display. There is one more thing missing, we should provide capability to clear the local framebuffer or clear the screen (i.e clear the framebuffer and push it to the devices).

The clear_buffer() function simply fills the framebuffer with zeros, effectively turning off all pixels in memory. This is useful when you want to prepare a clean slate for drawing new content.

```rust
pub fn clear_buffer(&mut self) {
    self.framebuffer.fill(0);
}
```

The clear_screen() function does two things: it clears the framebuffer and immediately flushes the changes to the display. This is what you want when you need to clear what's currently showing on the LED matrix.

```rust
pub fn clear_screen(&mut self) -> Result<()> {
    self.clear_buffer();
    self.flush()
}
```


## The completed Project

The completed project, which implements the embedded-graphics with the max7219-driver, is available here: 

[https://github.com/ImplFerris/max7219-eg](https://github.com/ImplFerris/max7219-eg)

## Tests

```rust
#[test]
fn test_clear_buffer() {
    let mut spi = SpiMock::new(&[]); // No SPI interaction
    let driver = Max7219::new(&mut spi);
    let mut matrix = SingleMatrix::from_driver(driver).unwrap();

    // Modify the buffer
    matrix.framebuffer[0] = 1;
    matrix.framebuffer[10] = 1;
    matrix.framebuffer[63] = 1;
    assert_ne!(matrix.framebuffer, [0u8; 64]);

    matrix.clear_buffer();

    assert_eq!(matrix.framebuffer, [0u8; 64]);
    spi.done();
}

#[test]
fn test_clear_screen() {
    // All digits 0..7 will be written with 0x00 for a single device
    let mut expected_transactions = Vec::new();
    for row in 0..8 {
        let digit_register = Register::try_digit(row).unwrap();
        expected_transactions.push(Transaction::transaction_start());
        expected_transactions.push(Transaction::write_vec(vec![digit_register.addr(), 0x00]));
        expected_transactions.push(Transaction::transaction_end());
    }

    let mut spi = SpiMock::new(&expected_transactions);
    let driver = Max7219::new(&mut spi);
    let mut matrix = SingleMatrix::from_driver(driver).unwrap();

    // Modify the buffer
    matrix.framebuffer[5] = 1;
    matrix.framebuffer[15] = 1;

    assert_ne!(matrix.framebuffer, [0u8; 64]);

    let result = matrix.clear_screen();
    assert!(result.is_ok());
    assert_eq!(matrix.framebuffer, [0u8; 64]);
    spi.done();
}
```
