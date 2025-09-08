# Non-volatile memory

Some RTCs include built-in non-volatile memory (NVRAM) for storing user data that persists even when the main power is lost. For example, the DS1307 has 56 bytes of NVRAM, while the DS3231 has none. This memory is separate from the timekeeping registers and can store configuration settings, calibration data, or any application-specific information.

The `RtcNvram` trait provides a simple interface for reading and writing this memory. It uses byte-level addressing with an offset parameter, allowing you to access any portion of the available NVRAM. The trait also includes a nvram_size() method so applications can determine how much memory is available.

## The full code for the nvram.rs module

```rust

/// RTC with non-volatile memory (NVRAM/SRAM) access
pub trait RtcNvram: Rtc {
    /// Read data from NVRAM starting at the given offset
    ///
    /// # Parameters
    /// * `offset` - NVRAM offset (0 = first NVRAM byte, up to device-specific max)
    /// * `buffer` - Buffer to store the read data
    ///
    /// # Returns
    /// * `Ok(())` on success
    /// * `Err(Self::Error)` if offset or length is invalid, or read fails
    fn read_nvram(&mut self, offset: u8, buffer: &mut [u8]) -> Result<(), Self::Error>;

    /// Write data to NVRAM starting at the given offset
    ///
    /// # Parameters
    /// * `offset` - NVRAM offset (0 = first NVRAM byte, up to device-specific max)
    /// * `data` - Data to write to NVRAM
    ///
    /// # Returns
    /// * `Ok(())` on success
    /// * `Err(Self::Error)` if offset or length is invalid, or write fails
    fn write_nvram(&mut self, offset: u8, data: &[u8]) -> Result<(), Self::Error>;

    /// Get the size of available NVRAM in bytes
    ///
    /// # Returns
    /// Total NVRAM size (e.g., 56 for DS1307, 0 for DS3231)
    fn nvram_size(&self) -> u16;
}
```
