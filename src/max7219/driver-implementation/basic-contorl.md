# Basic control

Now, we are going to define functions that use our low-level SPI methods to control the MAX7219 chip by writing the right values to specific registers. They make it easy to perform common tasks like turning devices on or off, testing the display, clearing it, or setting limits and brightness.

## Power On and Power Off All Devices

These functions let us power on or power off all the MAX7219 devices in the daisy chain at once. They use the write_all_registers function to send the Shutdown command to every device simultaneously.

```rust
pub fn power_on(&mut self) -> Result<()> {
    let ops = [(Register::Shutdown, 0x01); MAX_DISPLAYS];

    self.write_all_registers(&ops[..self.device_count])
}
pub fn power_off(&mut self) -> Result<()> {
    let ops = [(Register::Shutdown, 0x00); MAX_DISPLAYS];

    self.write_all_registers(&ops[..self.device_count])
}
```

## Power On and Power Off a Single Device
These functions target just one device in the chain, turning it on or off by writing the Shutdown register for that device only.

```rust
pub fn power_on_device(&mut self, device_index: usize) -> Result<()> {
    self.write_device_register(device_index, Register::Shutdown, 0x01)
}

pub fn power_off_device(&mut self, device_index: usize) -> Result<()> {
    self.write_device_register(device_index, Register::Shutdown, 0x00)
}
```

## Enable or Disable Display Test Mode

These allow you to enable or disable the test mode that lights up all segments of the display. You can test one device or all devices at once.


```rust
pub fn test_device(&mut self, device_index: usize, enable: bool) -> Result<()> {
        let data = if enable { 0x01 } else { 0x00 };
        self.write_device_register(device_index, Register::DisplayTest, data)
}

pub fn test_all(&mut self, enable: bool) -> Result<()> {
    let data = if enable { 0x01 } else { 0x00 };
    let ops: [(Register, u8); MAX_DISPLAYS] = [(Register::DisplayTest, data); MAX_DISPLAYS];
    self.write_all_registers(&ops[..self.device_count])
}
```

## Clear the Display

These functions clear the display by writing zero to all digit registers. You can clear a single device or all devices.

```rust
pub fn clear_display(&mut self, device_index: usize) -> Result<()> {
    for digit_register in Register::digits() {
        self.write_device_register(device_index, digit_register, 0x00)?;
    }
    Ok(())
}

pub fn clear_all(&mut self) -> Result<()> {
    for digit_register in Register::digits() {
        let ops = [(digit_register, 0x00); MAX_DISPLAYS];
        self.write_all_registers(&ops[..self.device_count])?;
    }

    Ok(())
}
```

## Set Intensity (Brightness)

You can control the brightness of a single device or all devices using these functions. Valid intensity values are from 0 to 15 (0x0F). Values outside this range return an error.

```rust
pub fn set_intensity(&mut self, device_index: usize, intensity: u8) -> Result<()> {
    if intensity > 0x0F {
        return Err(Error::InvalidIntensity);
    }
    self.write_device_register(device_index, Register::Intensity, intensity)
}

pub fn set_intensity_all(&mut self, intensity: u8) -> Result<()> {
    let ops = [(Register::Intensity, intensity); MAX_DISPLAYS];
    self.write_all_registers(&ops[..self.device_count])
}
```


## Set Scan Limit (Digits Visible)

The scan limit controls how many digits the display will show (from 1 to 8). These functions set the scan limit for a single device or all devices. The driver validates the input and returns an error if it’s out of range. This is mainly for the 7-segment display.

```rust
pub fn set_device_scan_limit(&mut self, device_index: usize, limit: u8) -> Result<()> {
    if !(1..=8).contains(&limit) {
        return Err(Error::InvalidScanLimit);
    }

    self.write_device_register(device_index, Register::ScanLimit, limit - 1)
}

pub fn set_scan_limit_all(&mut self, limit: u8) -> Result<()> {
    if !(1..=8).contains(&limit) {
        return Err(Error::InvalidScanLimit);
    }
    let val = limit - 1;
    let ops: [(Register, u8); MAX_DISPLAYS] = [(Register::ScanLimit, val); MAX_DISPLAYS];
    self.write_all_registers(&ops[..self.device_count])
}

```


## Tests

We will add tests to make sure the basic control functions work correctly and send the right SPI commands. These tests check turning power on and off for all devices or just one device, handling invalid device indexes, enabling and disabling test mode, setting scan limits and brightness (intensity) with proper input validation, and clearing the display. 

Each test uses the embedded-hal mock SPI and sets expectations for SPI transactions, including marking the start and end of each transaction and verifying the exact bytes sent.

```rust
#[test]
fn test_power_on() {
    let expected_transactions = [
        Transaction::transaction_start(),
        Transaction::write_vec(vec![Register::Shutdown.addr(), 0x01]),
        Transaction::transaction_end(),
    ];
    let mut spi = SpiMock::new(&expected_transactions);
    let mut driver = Max7219::new(&mut spi);

    driver.power_on().expect("Power on should succeed");
    spi.done();
}

#[test]
fn test_power_off() {
    let expected_transactions = [
        Transaction::transaction_start(),
        Transaction::write_vec(vec![Register::Shutdown.addr(), 0x00]),
        Transaction::transaction_end(),
    ];
    let mut spi = SpiMock::new(&expected_transactions);
    let mut driver = Max7219::new(&mut spi);

    driver.power_off().expect("Power off should succeed");
    spi.done();
}

#[test]
fn test_power_on_device() {
    let expected_transactions = [
        Transaction::transaction_start(),
        Transaction::write_vec(vec![Register::Shutdown.addr(), 0x01]),
        Transaction::transaction_end(),
    ];
    let mut spi = SpiMock::new(&expected_transactions);
    let mut driver = Max7219::new(&mut spi);

    driver
        .power_on_device(0)
        .expect("Power on display should succeed");
    spi.done();
}

// Test with multiple devices - power_on
#[test]
fn test_power_on_multiple_devices() {
    let device_count = 3;

    let expected_transactions = [
        Transaction::transaction_start(),
        Transaction::write_vec(vec![
            Register::Shutdown.addr(),
            0x01,
            Register::Shutdown.addr(),
            0x01,
            Register::Shutdown.addr(),
            0x01,
        ]),
        Transaction::transaction_end(),
    ];
    let mut spi = SpiMock::new(&expected_transactions);
    let mut driver = Max7219::new(&mut spi)
        .with_device_count(device_count)
        .expect("Should accept valid count");

    driver.power_on().expect("Power on should succeed");
    spi.done();
}

#[test]
fn test_power_off_device() {
    let expected_transactions = [
        Transaction::transaction_start(),
        Transaction::write_vec(vec![
            // For 4 devices
            Register::NoOp.addr(),
            0x00,
            Register::NoOp.addr(),
            0x00,
            Register::Shutdown.addr(),
            0x00,
            Register::NoOp.addr(),
            0x00,
        ]),
        Transaction::transaction_end(),
    ];
    let mut spi = SpiMock::new(&expected_transactions);
    let mut driver = Max7219::new(&mut spi)
        .with_device_count(4)
        .expect("a valid device count");

    driver
        .power_off_device(2)
        .expect("Power off display should succeed");
    spi.done();
}

#[test]
fn test_power_device_invalid_index() {
    let mut spi = SpiMock::new(&[]);
    let mut driver = Max7219::new(&mut spi).with_device_count(1).unwrap();

    let result = driver.power_on_device(1);
    assert_eq!(result, Err(Error::InvalidDeviceIndex));

    let result = driver.power_off_device(1);
    assert_eq!(result, Err(Error::InvalidDeviceIndex));
    spi.done();
}

#[test]
fn test_test_all_enable() {
    let expected_transactions = [
        Transaction::transaction_start(),
        Transaction::write_vec(vec![
            Register::DisplayTest.addr(),
            0x01,
            Register::DisplayTest.addr(),
            0x01,
            Register::DisplayTest.addr(),
            0x01,
            Register::DisplayTest.addr(),
            0x01,
        ]),
        Transaction::transaction_end(),
    ];
    let mut spi = SpiMock::new(&expected_transactions);
    let mut driver = Max7219::new(&mut spi)
        .with_device_count(4)
        .expect("valid device count");

    driver
        .test_all(true)
        .expect("Test all enable should succeed");
    spi.done();
}

#[test]
fn test_test_all_disable() {
    let expected_transactions = [
        Transaction::transaction_start(),
        Transaction::write_vec(vec![Register::DisplayTest.addr(), 0x00]),
        Transaction::transaction_end(),
    ];
    let mut spi = SpiMock::new(&expected_transactions);
    let mut driver = Max7219::new(&mut spi);

    driver
        .test_all(false)
        .expect("Test all disable should succeed");
    spi.done();
}

#[test]
fn test_set_scan_limit_all_valid() {
    let limit = 4;
    let expected_transactions = [
        Transaction::transaction_start(),
        Transaction::write_vec(vec![Register::ScanLimit.addr(), limit - 1]),
        Transaction::transaction_end(),
    ];
    let mut spi = SpiMock::new(&expected_transactions);
    let mut driver = Max7219::new(&mut spi);

    driver
        .set_scan_limit_all(limit)
        .expect("Set scan limit should succeed");
    spi.done();
}

#[test]
fn test_set_scan_limit_all_invalid_low() {
    let mut spi = SpiMock::new(&[]);
    let mut driver = Max7219::new(&mut spi);

    let result = driver.set_scan_limit_all(0);
    assert_eq!(result, Err(Error::InvalidScanLimit));
    spi.done();
}

#[test]
fn test_set_scan_limit_all_invalid_high() {
    let mut spi = SpiMock::new(&[]); // No transactions expected for invalid input
    let mut driver = Max7219::new(&mut spi);

    let result = driver.set_scan_limit_all(9);
    assert_eq!(result, Err(Error::InvalidScanLimit));
    spi.done();
}

#[test]
fn test_clear_display() {
    let mut expected_transactions = Vec::new();
    for digit_register in Register::digits() {
        expected_transactions.push(Transaction::transaction_start());
        expected_transactions.push(Transaction::write_vec(vec![digit_register.addr(), 0x00]));
        expected_transactions.push(Transaction::transaction_end());
    }

    let mut spi = SpiMock::new(&expected_transactions);
    let mut driver = Max7219::new(&mut spi);

    driver
        .clear_display(0)
        .expect("Clear display should succeed");
    spi.done();
}

#[test]
fn test_clear_display_invalid_index() {
    let mut spi = SpiMock::new(&[]); // No transactions expected for invalid index
    let mut driver = Max7219::new(&mut spi)
        .with_device_count(1)
        .expect("valid device count");

    let result = driver.clear_display(1);
    assert_eq!(result, Err(Error::InvalidDeviceIndex));
    spi.done();
}

#[test]
fn test_set_intensity_valid() {
    let device_index = 0;
    let intensity = 0x0A;
    let expected_transactions = [
        Transaction::transaction_start(),
        Transaction::write_vec(vec![Register::Intensity.addr(), intensity]),
        Transaction::transaction_end(),
    ];
    let mut spi = SpiMock::new(&expected_transactions);
    let mut driver = Max7219::new(&mut spi);

    driver
        .set_intensity(device_index, intensity)
        .expect("Set intensity should succeed");
    spi.done();
}

#[test]
fn test_set_intensity_invalid() {
    let mut spi = SpiMock::new(&[]); // No transactions expected for invalid input
    let mut driver = Max7219::new(&mut spi);

    let result = driver.set_intensity(0, 0x10); // Invalid intensity > 0x0F
    assert_eq!(result, Err(Error::InvalidIntensity));
    spi.done();
}

#[test]
fn test_test_device_enable_disable() {
    let expected_transactions = [
        Transaction::transaction_start(),
        Transaction::write_vec(vec![Register::DisplayTest.addr(), 0x01]),
        Transaction::transaction_end(),
        Transaction::transaction_start(),
        Transaction::write_vec(vec![Register::DisplayTest.addr(), 0x00]),
        Transaction::transaction_end(),
    ];
    let mut spi = SpiMock::new(&expected_transactions);
    let mut driver = Max7219::new(&mut spi);

    driver
        .test_device(0, true)
        .expect("Enable test mode failed");
    driver
        .test_device(0, false)
        .expect("Disable test mode failed");
    spi.done();
}

#[test]
fn test_set_device_scan_limit_valid() {
    let scan_limit = 4;

    let expected_transactions = [
        Transaction::transaction_start(),
        Transaction::write_vec(vec![Register::ScanLimit.addr(), scan_limit - 1]),
        Transaction::transaction_end(),
    ];
    let mut spi = SpiMock::new(&expected_transactions);
    let mut driver = Max7219::new(&mut spi);

    driver
        .set_device_scan_limit(0, scan_limit)
        .expect("Scan limit set failed");
    spi.done();
}

#[test]
fn test_set_device_scan_limit_invalid() {
    let mut spi = SpiMock::new(&[]);
    let mut driver = Max7219::new(&mut spi);

    let result = driver.set_device_scan_limit(0, 0); // invalid: below range
    assert_eq!(result, Err(Error::InvalidScanLimit));

    let result = driver.set_device_scan_limit(0, 9); // invalid: above range
    assert_eq!(result, Err(Error::InvalidScanLimit));
    spi.done();
}

#[test]
fn test_set_intensity_all() {
    let intensity = 0x05;
    let expected_transactions = [
        Transaction::transaction_start(),
        Transaction::write_vec(vec![
            Register::Intensity.addr(),
            intensity,
            Register::Intensity.addr(),
            intensity,
        ]),
        Transaction::transaction_end(),
    ];
    let mut spi = SpiMock::new(&expected_transactions);
    let mut driver = Max7219::new(&mut spi)
        .with_device_count(2)
        .expect("valid count");

    driver
        .set_intensity_all(intensity)
        .expect("Set intensity all failed");
    spi.done();
}
```
