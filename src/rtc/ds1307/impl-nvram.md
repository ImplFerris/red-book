# impl RtcNvram

Let's implement the `RtcNvram` trait to provide access to the DS1307's non-volatile memory.

```rust
impl<I2C> RtcNvram for Ds1307<I2C>
where
    I2C: embedded_hal::i2c::I2c,
{
    fn read_nvram(&mut self, offset: u8, buffer: &mut [u8]) -> Result<(), Self::Error> {
        if buffer.is_empty() {
            return Ok(());
        }

        self.validate_nvram_bounds(offset, buffer.len())?;

        let nvram_addr = NVRAM_START + offset;
        self.read_bytes_at_address(nvram_addr, buffer)?;

        Ok(())
    }

    fn write_nvram(&mut self, offset: u8, data: &[u8]) -> Result<(), Self::Error> {
        if data.is_empty() {
            return Ok(());
        }

        self.validate_nvram_bounds(offset, data.len())?;

        let mut buffer = [0u8; MAX_NVRAM_WRITE];
        buffer[0] = NVRAM_START + offset;
        buffer[1..data.len() + 1].copy_from_slice(data);

        self.write_raw_bytes(&buffer[..data.len() + 1])?;

        Ok(())
    }

    fn nvram_size(&self) -> u16 {
        NVRAM_SIZE as u16
    }
}
```

### Method to get NVRAM Size
In the nvram_size implementation, we just return the NVRAM_SIZE constant that we declared in the registers module.

### Reading data from NVRAM

In the read_nvram method, we first check if the buffer is empty and return early if there's no space to store the read data.

We then do the bounds check to ensure the offset is within the NVRAM memory layout and that offset + buffer length stays within the memory boundaries.

We calculate the actual NVRAM address by adding the NVRAM starting address with the given offset. Finally we use the read_bytes_at_address method to read the data in one burst operation.

### Writing data to NVRAM

For the write_nvram method, we follow a similar approach but prepare the data for a burst write operation.

We create a buffer with the starting address followed by the actual data to write, then use write_raw_bytes to send everything in one I2C transaction.


## The full code for the nvram module (nvram.rs)

```rust
//! DS1307 NVRAM Support
//!
//! This module provides an implementation of the [`RtcNvram`] trait for the
//! [`Ds1307`] real-time clock (RTC).

pub use rtc_hal::nvram::RtcNvram;

use crate::{
    Ds1307,
    registers::{MAX_NVRAM_WRITE, NVRAM_SIZE, NVRAM_START},
};

impl<I2C> RtcNvram for Ds1307<I2C>
where
    I2C: embedded_hal::i2c::I2c,
{
    /// Read data from DS1307 NVRAM.
    ///
    /// - `offset`: starting NVRAM address (0..55)
    /// - `buffer`: output buffer to store the read data
    ///
    /// Performs a sequential read starting at `NVRAM_START + offset`.
    fn read_nvram(&mut self, offset: u8, buffer: &mut [u8]) -> Result<(), Self::Error> {
        if buffer.is_empty() {
            return Ok(());
        }

        self.validate_nvram_bounds(offset, buffer.len())?;

        let nvram_addr = NVRAM_START + offset;
        self.read_bytes_at_address(nvram_addr, buffer)?;

        Ok(())
    }

    /// Write data into DS1307 NVRAM.
    ///
    /// - `offset`: starting NVRAM address (0..55)
    /// - `data`: slice containing data to write
    ///
    /// Uses either single-byte write or burst write depending on length.
    fn write_nvram(&mut self, offset: u8, data: &[u8]) -> Result<(), Self::Error> {
        if data.is_empty() {
            return Ok(());
        }

        self.validate_nvram_bounds(offset, data.len())?;

        // Burst write
        let mut buffer = [0u8; MAX_NVRAM_WRITE];
        buffer[0] = NVRAM_START + offset;
        buffer[1..data.len() + 1].copy_from_slice(data);

        self.write_raw_bytes(&buffer[..data.len() + 1])?;

        Ok(())
    }

    /// Return the size of DS1307 NVRAM in bytes (56).
    fn nvram_size(&self) -> u16 {
        NVRAM_SIZE as u16
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::Ds1307;
    use crate::error::Error;
    use embedded_hal_mock::eh1::i2c::{Mock as I2cMock, Transaction as I2cTransaction};
    use rtc_hal::nvram::RtcNvram;

    const DS1307_ADDR: u8 = 0x68;
    const NVRAM_START: u8 = 0x08;
    const NVRAM_SIZE: u8 = 56;

    #[test]
    fn test_nvram_size() {
        let i2c_mock = I2cMock::new(&[]);
        let ds1307 = Ds1307::new(i2c_mock);

        assert_eq!(ds1307.nvram_size(), NVRAM_SIZE as u16);

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_validate_nvram_bounds_valid() {
        let i2c_mock = I2cMock::new(&[]);
        let ds1307 = Ds1307::new(i2c_mock);

        // Test valid cases
        assert!(ds1307.validate_nvram_bounds(0, 1).is_ok());
        assert!(ds1307.validate_nvram_bounds(0, 56).is_ok());
        assert!(ds1307.validate_nvram_bounds(55, 1).is_ok());
        assert!(ds1307.validate_nvram_bounds(10, 46).is_ok());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_validate_nvram_bounds_invalid_offset() {
        let i2c_mock = I2cMock::new(&[]);
        let ds1307 = Ds1307::new(i2c_mock);

        // Test invalid offset
        assert!(matches!(
            ds1307.validate_nvram_bounds(56, 1),
            Err(Error::NvramOutOfBounds)
        ));
        assert!(matches!(
            ds1307.validate_nvram_bounds(100, 1),
            Err(Error::NvramOutOfBounds)
        ));

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_validate_nvram_bounds_invalid_length() {
        let i2c_mock = I2cMock::new(&[]);
        let ds1307 = Ds1307::new(i2c_mock);

        // Test length that goes beyond NVRAM
        assert!(matches!(
            ds1307.validate_nvram_bounds(0, 57),
            Err(Error::NvramOutOfBounds)
        ));
        assert!(matches!(
            ds1307.validate_nvram_bounds(55, 2),
            Err(Error::NvramOutOfBounds)
        ));
        assert!(matches!(
            ds1307.validate_nvram_bounds(10, 50),
            Err(Error::NvramOutOfBounds)
        ));

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_read_nvram_single_byte() {
        let expectations = vec![I2cTransaction::write_read(
            DS1307_ADDR,
            vec![NVRAM_START + 10], // Read from NVRAM offset 10
            vec![0xAB],
        )];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let mut buffer = [0u8; 1];
        let result = ds1307.read_nvram(10, &mut buffer);
        assert!(result.is_ok());
        assert_eq!(buffer[0], 0xAB);

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_read_nvram_multiple_bytes() {
        let expectations = vec![I2cTransaction::write_read(
            DS1307_ADDR,
            vec![NVRAM_START + 5], // Read from NVRAM offset 5
            vec![0x01, 0x02, 0x03, 0x04, 0x05],
        )];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let mut buffer = [0u8; 5];
        let result = ds1307.read_nvram(5, &mut buffer);
        assert!(result.is_ok());
        assert_eq!(buffer, [0x01, 0x02, 0x03, 0x04, 0x05]);

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_read_nvram_from_start() {
        let expectations = vec![I2cTransaction::write_read(
            DS1307_ADDR,
            vec![NVRAM_START], // Read from beginning of NVRAM
            vec![0xFF, 0xEE],
        )];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let mut buffer = [0u8; 2];
        let result = ds1307.read_nvram(0, &mut buffer);
        assert!(result.is_ok());
        assert_eq!(buffer, [0xFF, 0xEE]);

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_read_nvram_from_end() {
        let expectations = vec![I2cTransaction::write_read(
            DS1307_ADDR,
            vec![NVRAM_START + 55], // Read from last NVRAM byte
            vec![0x42],
        )];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let mut buffer = [0u8; 1];
        let result = ds1307.read_nvram(55, &mut buffer);
        assert!(result.is_ok());
        assert_eq!(buffer[0], 0x42);

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_read_nvram_full_size() {
        let expected_data = vec![NVRAM_START];
        let mut response_data = Vec::new();
        for i in 0..NVRAM_SIZE {
            response_data.push(i);
        }

        let expectations = vec![I2cTransaction::write_read(
            DS1307_ADDR,
            expected_data,
            response_data.clone(),
        )];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let mut buffer = [0u8; NVRAM_SIZE as usize];
        let result = ds1307.read_nvram(0, &mut buffer);
        assert!(result.is_ok());
        assert_eq!(buffer.to_vec(), response_data);

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_read_nvram_empty_buffer() {
        let i2c_mock = I2cMock::new(&[]);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let mut buffer = [];
        let result = ds1307.read_nvram(0, &mut buffer);
        assert!(result.is_ok());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_read_nvram_out_of_bounds_offset() {
        let i2c_mock = I2cMock::new(&[]);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let mut buffer = [0u8; 1];
        let result = ds1307.read_nvram(56, &mut buffer);
        assert!(matches!(result, Err(Error::NvramOutOfBounds)));

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_read_nvram_out_of_bounds_length() {
        let i2c_mock = I2cMock::new(&[]);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let mut buffer = [0u8; 2];
        let result = ds1307.read_nvram(55, &mut buffer);
        assert!(matches!(result, Err(Error::NvramOutOfBounds)));

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_read_nvram_i2c_error() {
        let expectations = vec![
            I2cTransaction::write_read(DS1307_ADDR, vec![NVRAM_START], vec![0x00])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let mut buffer = [0u8; 1];
        let result = ds1307.read_nvram(0, &mut buffer);
        assert!(result.is_err());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_write_nvram_single_byte() {
        let expectations = vec![I2cTransaction::write(
            DS1307_ADDR,
            vec![NVRAM_START + 10, 0xCD], // Write to NVRAM offset 10
        )];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let data = [0xCD];
        let result = ds1307.write_nvram(10, &data);
        assert!(result.is_ok());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_write_nvram_multiple_bytes() {
        let expectations = vec![I2cTransaction::write(
            DS1307_ADDR,
            vec![NVRAM_START + 5, 0x10, 0x20, 0x30, 0x40], // Write to NVRAM offset 5
        )];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let data = [0x10, 0x20, 0x30, 0x40];
        let result = ds1307.write_nvram(5, &data);
        assert!(result.is_ok());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_write_nvram_to_start() {
        let expectations = vec![I2cTransaction::write(
            DS1307_ADDR,
            vec![NVRAM_START, 0xAA, 0xBB], // Write to beginning of NVRAM
        )];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let data = [0xAA, 0xBB];
        let result = ds1307.write_nvram(0, &data);
        assert!(result.is_ok());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_write_nvram_to_end() {
        let expectations = vec![I2cTransaction::write(
            DS1307_ADDR,
            vec![NVRAM_START + 55, 0x99], // Write to last NVRAM byte
        )];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let data = [0x99];
        let result = ds1307.write_nvram(55, &data);
        assert!(result.is_ok());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_write_nvram_full_size() {
        let mut expected_data = vec![NVRAM_START];
        let write_data: Vec<u8> = (0..NVRAM_SIZE).collect();
        expected_data.extend_from_slice(&write_data);

        let expectations = vec![I2cTransaction::write(DS1307_ADDR, expected_data)];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let result = ds1307.write_nvram(0, &write_data);
        assert!(result.is_ok());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_write_nvram_empty_data() {
        let i2c_mock = I2cMock::new(&[]);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let data = [];
        let result = ds1307.write_nvram(0, &data);
        assert!(result.is_ok());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_write_nvram_out_of_bounds_offset() {
        let i2c_mock = I2cMock::new(&[]);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let data = [0x42];
        let result = ds1307.write_nvram(56, &data);
        assert!(matches!(result, Err(Error::NvramOutOfBounds)));

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_write_nvram_out_of_bounds_length() {
        let i2c_mock = I2cMock::new(&[]);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let data = [0x42, 0x43];
        let result = ds1307.write_nvram(55, &data);
        assert!(matches!(result, Err(Error::NvramOutOfBounds)));

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_write_nvram_i2c_error() {
        let expectations = vec![
            I2cTransaction::write(DS1307_ADDR, vec![NVRAM_START, 0x42])
                .with_error(embedded_hal::i2c::ErrorKind::Other),
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let data = [0x42];
        let result = ds1307.write_nvram(0, &data);
        assert!(result.is_err());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_constants() {
        assert_eq!(NVRAM_START, 0x08);
        assert_eq!(NVRAM_SIZE, 56);
        assert_eq!(MAX_NVRAM_WRITE, 57);
    }

    #[test]
    fn test_nvram_boundary_conditions() {
        // Test reading/writing exactly at boundaries
        let expectations = vec![I2cTransaction::write_read(
            DS1307_ADDR,
            vec![NVRAM_START + 20],
            vec![
                0x11, 0x22, 0x33, 0x44, 0x55, 0x66, 0x77, 0x88, 0x99, 0xAA, 0xBB, 0xCC, 0xDD, 0xEE,
                0xFF, 0x00, 0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08, 0x09, 0x0A, 0x0B, 0x0C,
                0x0D, 0x0E, 0x0F, 0x10, 0x11, 0x12, 0x13, 0x14,
            ],
        )];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        // Read from offset 20 to the end (36 bytes)
        let mut buffer = [0u8; 36];
        let result = ds1307.read_nvram(20, &mut buffer);
        assert!(result.is_ok());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_write_nvram_burst_write_format() {
        // Test that the burst write format is correct
        let expectations = vec![I2cTransaction::write(
            DS1307_ADDR,
            vec![NVRAM_START + 15, 0x11, 0x22, 0x33, 0x44, 0x55, 0x66, 0x77],
        )];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let data = [0x11, 0x22, 0x33, 0x44, 0x55, 0x66, 0x77];
        let result = ds1307.write_nvram(15, &data);
        assert!(result.is_ok());

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }

    #[test]
    fn test_nvram_address_calculation() {
        // Test that NVRAM addresses are calculated correctly
        let expectations = vec![
            I2cTransaction::write_read(DS1307_ADDR, vec![0x08], vec![0x01]), // offset 0
            I2cTransaction::write_read(DS1307_ADDR, vec![0x10], vec![0x02]), // offset 8
            I2cTransaction::write_read(DS1307_ADDR, vec![0x20], vec![0x03]), // offset 24
            I2cTransaction::write_read(DS1307_ADDR, vec![0x3F], vec![0x04]), // offset 55
        ];

        let i2c_mock = I2cMock::new(&expectations);
        let mut ds1307 = Ds1307::new(i2c_mock);

        let mut buffer = [0u8; 1];

        // Test offset 0 -> address 0x08
        assert!(ds1307.read_nvram(0, &mut buffer).is_ok());
        assert_eq!(buffer[0], 0x01);

        // Test offset 8 -> address 0x10
        assert!(ds1307.read_nvram(8, &mut buffer).is_ok());
        assert_eq!(buffer[0], 0x02);

        // Test offset 24 -> address 0x20
        assert!(ds1307.read_nvram(24, &mut buffer).is_ok());
        assert_eq!(buffer[0], 0x03);

        // Test offset 55 -> address 0x3F
        assert!(ds1307.read_nvram(55, &mut buffer).is_ok());
        assert_eq!(buffer[0], 0x04);

        let mut i2c_mock = ds1307.release_i2c();
        i2c_mock.done();
    }
}
```
