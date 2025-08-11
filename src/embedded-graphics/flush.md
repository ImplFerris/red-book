# Flushing the Framebuffer to the Max7219 LED Matrix

We have successfully implemented embedded-graphics for the LedMatrix, and now we need to update the actual LED matrix hardware with the pixel data stored in our internal framebuffer. For this, we will define a method called "flush".

Required imports

```rust
use crate::registers::Register;
use crate::Result;
```

The flush method sends data row by row (from row 0 to row 7) to all daisy-chained devices. For each row, it prepares an array of SPI commands; one per device that represent that row's pixels packed into a single byte.

```rust
impl<SPI, const BUFFER_LENGTH: usize, const DEVICE_COUNT: usize>
    LedMatrix<SPI, BUFFER_LENGTH, DEVICE_COUNT>
where
    SPI: SpiDevice,
{
    // ...other functions like from_driver that we created before

    pub fn flush(&mut self) -> Result<()> {
        for (row, digit_register) in Register::digits().enumerate() {
            let mut ops = [(Register::NoOp, 0); DEVICE_COUNT];

            for (device_index, op) in ops.iter_mut().enumerate() {
                let buffer_start = device_index * 64 + row * 8;
                let mut packed_byte = 0b0000_0000;

                for col in 0..8 {
                    let pixel_index = buffer_start + col;
                    if pixel_index < self.framebuffer.len() && self.framebuffer[pixel_index] != 0 {
                        // bit 7 is leftmost pixel (Col 0) on the display
                        packed_byte |= 1 << (7 - col);
                    }
                }

                *op = (digit_register, packed_byte);
            }

            self.driver.write_all_registers(&ops[..DEVICE_COUNT])?;
        }
        Ok(())
    }
}
```

Let's walk through the code step-by-step:

## Batched SPI Commands

We send SPI commands in batches; one SPI transaction per row regardless of the number of devices. So, at the top-most loop, we iterate over the Digit Registers (from 0 to 7), representing each row of the display.

```rust
for (row, digit_register) in Register::digits().enumerate() {
```

We create an array named ops to hold SPI commands for each device. Initially, each command is set to a No-Op with a data byte of 0. The size of this array equals the number of connected devices.

```rust
let mut ops = [(Register::NoOp, 0); DEVICE_COUNT];
```

### Finding the Column Start Position for the Current Row

Next, we determine which columns correspond to the current row for each device by reading from the framebuffer. We loop through all devices and calculate the starting position in the framebuffer for each device and row.

<div style="text-align: center;">
  <a href="./images/framebuffer-max7219-deviec-indices.svg"><img style="display: block; margin: auto;" alt="Max7219 Devices and Framebuffer indices" src="./images/framebuffer-max7219-deviec-indices.svg"/></a>
  <figcaption style="font-style: italic; margin-top: 8px; color: #555;">
    Figure 1: 4 daisy-chained Max7219 devices with corresponding framebuffer indices.
  </figcaption>
</div> 

We calculate buffer_start as:

```rust
let buffer_start = device_index * 64 + row * 8;
```

This calculation works because each device's framebuffer occupies 64 bytes (8 rows × 8 columns). Multiplying device_index by 64 jumps to that device's start, then adding row * 8 moves to the correct row within that device.

For example, if we are processing the second row (row index 1) of the third device (device_index 2), the buffer start is:

```rust
2 * 64 + 1 * 8 = 128 + 8 = 136
```

You can verify this position against the illustration above, where the buffer_start matches the row and device's location in the framebuffer.

## Build Data Packet

Now that we know the starting column position for the current row and device, we need to create the data packet representing which pixels in this row are turned on or off.

For each device, we extract the 8 pixels of the current row from the framebuffer. These 8 pixels are packed into a single byte, where bit 7 corresponds to the leftmost pixel (column 0), and bit 0 corresponds to the rightmost pixel (column 7).

We start by initializing a byte with all bits set to zero. This byte will hold the pixel states for the 8 columns in the current row:

```rust
let mut packed_byte = 0b0000_0000; 
```

Next, we loop over each column (from 0 to 7). For each column, we calculate the pixel's index in the framebuffer by adding buffer_start + col. If the pixel at that index is ON (i.e non-zero), we set the corresponding bit in packed_byte. Because bit 7 is the leftmost pixel, the bit to set is 7 - col:

```rust
for col in 0..8 {
    let pixel_index = buffer_start + col;
    if pixel_index < self.framebuffer.len() && self.framebuffer[pixel_index] != 0 {
        // bit 7 is leftmost pixel (Col 0) on the display
        packed_byte |= 1 << (7 - col);
    }
}
```

For example, suppose the framebuffer values for a given row and device's columns are:  `[1, 0, 1, 0, 1, 0, 1, 0]`

This would be packed as:

| Bit 7 | Bit 6 | Bit 5 | Bit 4 | Bit 3 | Bit 2 | Bit 1 | Bit 0 |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| 1     | 0     | 1     | 0     | 1     | 0     | 1     | 0     |

Resulting in a binary value of 0b10101010.

## SPI Operations

Now that we have the digit register (row), device index, and packed data byte representing the pixels, we can prepare the SPI operation array to send this data to the devices.

For each device, we create a tuple of (digit_register, packed_byte):

```rust
*op = (digit_register, packed_byte);
```

After processing all devices for the current row, the ops array will contain an entry for each device, something like this (assuming 4 devices):

```rust
[
    (digit_register, packed_byte_device_0),
    (digit_register, packed_byte_device_1),
    (digit_register, packed_byte_device_2),
    (digit_register, packed_byte_device_3),
]
```

Finally, we send all the prepared operations for the current row in a single SPI transaction:

```rust
self.driver.write_all_registers(&ops[..DEVICE_COUNT])?;
```

This process is repeated for each row until the entire framebuffer is sent to the devices.

 
## Tests

```rust
#[test]
fn test_flush_single_device() {
    // We expect the flush to send 8 SPI transactions, one for each row (DIGIT0 to DIGIT7)
    // Only rows 0 and 7 have pixel data: 0b10101010 (columns 0,2,4,6 lit)
    // All other rows should be cleared (0b00000000)

    let mut expected_transactions = Vec::new();
    for (row, digit_register) in Register::digits().enumerate() {
        // For rows 0 and 7, the framebuffer will result in this pattern:
        // Columns 0, 2, 4, 6 are ON => bits 7, 5, 3, 1 set => 0b10101010
        let expected_byte = if row == 0 || row == 7 {
            0b10101010
        } else {
            0b00000000
        };

        // Each transaction sends [register, data] for that row
        expected_transactions.push(Transaction::transaction_start());
        expected_transactions.push(Transaction::write_vec(vec![
            digit_register.addr(),
            expected_byte,
        ]));
        expected_transactions.push(Transaction::transaction_end());
    }

    // Create the SPI mock with the expected sequence of writes
    let mut spi = SpiMock::new(&expected_transactions);
    let driver = Max7219::new(&mut spi);
    let mut matrix: LedMatrix<_> = LedMatrix::from_driver(driver).unwrap();

    // Set framebuffer values to light up alternating columns in row 0 and row 7
    // Row 0 corresponds to framebuffer indices 0 to 7
    matrix.framebuffer[0] = 1; // Column 0
    matrix.framebuffer[2] = 1; // Column 2
    matrix.framebuffer[4] = 1; // Column 4
    matrix.framebuffer[6] = 1; // Column 6

    // Each device's framebuffer is a flat array of 64 bytes: 8 rows * 8 columns
    // The layout is row-major: [row0[0..7], row1[0..7], ..., row7[0..7]]
    //
    // For a single device:
    //   framebuffer[ 0.. 7] => row 0
    //   framebuffer[ 8..15] => row 1
    //   framebuffer[16..23] => row 2
    //   framebuffer[24..31] => row 3
    //   framebuffer[32..39] => row 4
    //   framebuffer[40..47] => row 5
    //   framebuffer[48..55] => row 6
    //   framebuffer[56..63] => row 7 (last row)
    //
    // So to update row 7, we write to indices 56 to 63.
    matrix.framebuffer[56] = 1; // Column 0
    matrix.framebuffer[58] = 1; // Column 2
    matrix.framebuffer[60] = 1; // Column 4
    matrix.framebuffer[62] = 1; // Column 6

    // Call flush, which will convert framebuffer rows into bytes and send via SPI
    let result = matrix.flush();
    assert!(result.is_ok());

    spi.done();
}

#[test]
fn test_flush_multiple_devices() {
    const TEST_DEVICE_COUNT: usize = 4;
    const TEST_BUFF_LEN: usize = 256;

    let mut expected_transactions = Vec::new();

    for (row, digit_register) in Register::digits().enumerate() {
        expected_transactions.push(Transaction::transaction_start());

        // For each device, we write the register and data byte

        let ops_array = (0..TEST_DEVICE_COUNT)
            .flat_map(|device| {
                let expected_byte = match (row, device) {
                    (0, 0) => 0b10101000,
                    (7, 2) => 0b00010101,
                    _ => 0b0000_0000,
                };
                vec![digit_register.addr(), expected_byte]
            })
            .collect();

        expected_transactions.push(Transaction::write_vec(ops_array));

        expected_transactions.push(Transaction::transaction_end());
    }

    // Create SPI mock with expected transactions
    let mut spi = SpiMock::new(&expected_transactions);
    let driver = Max7219::new(&mut spi)
        .with_device_count(TEST_DEVICE_COUNT)
        .unwrap();

    let mut matrix: LedMatrix<_, TEST_BUFF_LEN, TEST_DEVICE_COUNT> =
        LedMatrix::from_driver(driver).unwrap();

    // Set pixels for device 0
    matrix.framebuffer[0] = 1; // row 0, col 0
    matrix.framebuffer[2] = 1; // row 0, col 2
    matrix.framebuffer[4] = 1; // row 0, col 4

    // Set pixels for device 2
    matrix.framebuffer[64 * 2 + 7 * 8 + 3] = 1; // row 7, col 3
    matrix.framebuffer[64 * 2 + 7 * 8 + 5] = 1; // row 7, col 5
    matrix.framebuffer[64 * 2 + 7 * 8 + 7] = 1; // row 7, col 7

    // Call flush, which converts framebuffer rows into bytes and sends via SPI
    let result = matrix.flush();
    assert!(result.is_ok());

    spi.done();
}
```
