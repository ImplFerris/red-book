# Digit and Decode Mode Functions

We will add functions to control individual digit segments and set decode modes for devices.

## Writing Raw Digit Data

The write_raw_digit function lets us send a raw 8-bit value directly to a specific digit register (DIG0 to DIG7) on a chosen device. This gives you low-level control over exactly which segments or LEDs light up.

```rust
pub fn write_raw_digit(&mut self, device_index: usize, digit: u8, value: u8) -> Result<()> {
    let digit_register = Register::try_digit(digit)?;
    self.write_device_register(device_index, digit_register, value)
}
```

The function takes three arguments: the device_index (which device in the chain, 0 is the closest to the MCU), the digit (0 to 7), and the raw value byte.

### How it works for 7-segment displays
A typical 7-segment digit looks like this:

```sh
    A
   ---
F |   | B
  |   |
   ---
E |   | C
  |   |
   ---   . DP
    D
```

Each bit in the byte you send corresponds to one segment or the decimal point (DP). The bit layout is:

| Bit     | 7  | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
| ------- | -- | - | - | - | - | - | - | - |
| Segment | DP | A | B | C | D | E | F | G |

For example, to show the number 1, you send 0b00110000, which lights segments B and C.

### How it works for 8x8 LED matrices

For an LED matrix, each digit register controls one row. Each bit in the byte controls one column from left to right.

For example, on a common FC-16 module, DIG0 is the top row, and bit 0 is the rightmost column. Sending 0b10101010 to DIG0 would light every other LED across the top row like this:

```sh
DIG0 -> Row 0: value = 0b10101010

Matrix:
          Columns
           7 6 5 4 3 2 1 0
         +----------------
     0   | 1 0 1 0 1 0 1 0
     1   | ...
     2   | ...
   ...   | ...
     7   | ...
```

> Note: Wiring and orientation vary between displays. Some modules map rows and columns differently. If your output looks flipped or rotated, you may need to adjust your digit or bit mapping.


## Setting Decode Mode

The decode mode controls how the MAX7219 interprets the data sent to digit registers.

You can set decode mode for a single device using set_device_decode_mode, or for all devices at once using set_decode_mode_all.

Decode mode is important especially for 7-segment displays since it tells the chip whether to interpret raw bits or BCD-encoded digits.

```rust
pub fn set_device_decode_mode(&mut self, device_index: usize, mode: DecodeMode) -> Result<()> {
    self.write_device_register(device_index, Register::DecodeMode, mode as u8)
}

pub fn set_decode_mode_all(&mut self, mode: DecodeMode) -> Result<()> {
    let byte = mode as u8;
    let ops: [(Register, u8); MAX_DISPLAYS] = [(Register::DecodeMode, byte); MAX_DISPLAYS];
    self.write_all_registers(&ops[..self.device_count])
}
```

## Tests

We will add tests to verify the write_raw_digit and set_device_decode_mode functions.

The first test checks that writing a raw value to a digit register on a valid device sends the correct SPI commands. It uses device index 0 (closest device), digit 3, and a sample data byte 0b10101010. The SPI mock expects the correct register and data bytes wrapped in a transaction.

The second test verifies input validation by attempting to write to an invalid digit (8), which should return an InvalidDigit error without any SPI activity.

The third test confirms that setting the decode mode on a device sends the appropriate SPI command with the selected decode mode value.

```rust
#[test]
fn test_write_raw_digit() {
    let device_index = 0;
    let digit = 3;
    let data = 0b10101010;
    let expected_transactions = [
        Transaction::transaction_start(),
        Transaction::write_vec(vec![Register::Digit3.addr(), data]),
        Transaction::transaction_end(),
    ];
    let mut spi = SpiMock::new(&expected_transactions);
    let mut driver = Max7219::new(&mut spi);

    driver
        .write_raw_digit(device_index, digit, data)
        .expect("Write raw digit should succeed");
    spi.done();
}

#[test]
fn test_write_raw_digit_invalid_digit() {
    let mut spi = SpiMock::new(&[]); // No transactions expected for invalid digit
    let mut driver = Max7219::new(&mut spi);

    let result = driver.write_raw_digit(0, 8, 0x00); // Digit 8 is invalid

    assert_eq!(result, Err(Error::InvalidDigit));

    spi.done();
}

#[test]
fn test_set_device_decode_mode() {
    let mode = DecodeMode::Digits0To3;
    let expected_transactions = [
        Transaction::transaction_start(),
        Transaction::write_vec(vec![Register::DecodeMode.addr(), mode.value()]),
        Transaction::transaction_end(),
    ];
    let mut spi = SpiMock::new(&expected_transactions);
    let mut driver = Max7219::new(&mut spi);

    driver
        .set_device_decode_mode(0, mode)
        .expect("Set decode mode failed");
    spi.done();
}
```





