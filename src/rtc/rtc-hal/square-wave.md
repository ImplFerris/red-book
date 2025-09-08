# Square Wave Trait 

Many RTCs support a feature called "square wave" output through a hardware pin. You can set different square wave frequencies like 1Hz, 1024 Hz, 32kHz and so on. The available frequencies depend on the specific RTC chip.

The square wave output is commonly used for generating precise periodic interrupts. For example, you can set the RTC to output a 1Hz signal and use it to interrupt your microcontroller every second for updating displays or performing periodic tasks without constantly polling the RTC. 

We will create SquareWaveFreq enum with the standard frequencies supported by most RTC chips. The Custom variant allows drivers to support chip-specific frequencies not covered by the standard options.

```rust
/// Square wave output frequencies
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum SquareWaveFreq {
    /// 1 Hz
    Hz1,
    /// 1024 Hz (1.024 kHz)
    Hz1024,
    /// 4096 Hz (4.096 kHz)
    Hz4096,
    /// 8192 Hz (8.192 kHz)
    Hz8192,
    /// 32768 Hz (32.768 kHz)
    Hz32768,
    /// Custom frequency (if supported by device)
    Custom(u32),
}
```

Our square wave trait has four methods. The start_square_wave() method sets the frequency and starts the square wave output in one operation. The enable_square_wave() and disable_square_wave() methods allow you to turn the output on and off without changing the frequency. Finally, set_square_wave_frequency() lets you change the frequency without enabling or disabling the output.

```rust
/// Square wave functionality trait
pub trait SquareWave: Rtc {
    /// Configure Frequency and enable square wave
    fn start_square_wave(&mut self, freq: SquareWaveFreq) -> Result<(), Self::Error>;

    /// Enable square wave output
    fn enable_square_wave(&mut self) -> Result<(), Self::Error>;

    /// Disable square wave output
    fn disable_square_wave(&mut self) -> Result<(), Self::Error>;

    /// Set the frequency (without enabling/disabling)
    fn set_square_wave_frequency(&mut self, freq: SquareWaveFreq) -> Result<(), Self::Error>;
}
```

We'll also provide utility methods for the SquareWaveFreq enum. The to_hz() method converts it to the actual frequency value in Hz, while from_hz() converts a Hz value back into the appropriate enum variant.

```rust

impl SquareWaveFreq {
    /// Get frequency value in Hz
    pub fn to_hz(&self) -> u32 {
        match self {
            Self::Hz1 => 1,
            Self::Hz1024 => 1024,
            Self::Hz4096 => 4096,
            Self::Hz8192 => 8192,
            Self::Hz32768 => 32768,
            Self::Custom(freq) => *freq,
        }
    }

    /// Create from Hz value
    pub fn from_hz(hz: u32) -> Self {
        match hz {
            1 => Self::Hz1,
            1024 => Self::Hz1024,
            4096 => Self::Hz4096,
            8192 => Self::Hz8192,
            32768 => Self::Hz32768,
            other => Self::Custom(other),
        }
    }
}
```

## The full code for the square_wave.rs module

```rust
//! Traits for Square Wave control

use crate::rtc::Rtc;

/// Square wave output frequencies
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum SquareWaveFreq {
    /// 1 Hz
    Hz1,
    /// 1024 Hz (1.024 kHz)
    Hz1024,
    /// 4096 Hz (4.096 kHz)
    Hz4096,
    /// 8192 Hz (8.192 kHz)
    Hz8192,
    /// 32768 Hz (32.768 kHz)
    Hz32768,
    /// Custom frequency (if supported by device)
    Custom(u32),
}

impl SquareWaveFreq {
    /// Get frequency value in Hz
    pub fn to_hz(&self) -> u32 {
        match self {
            Self::Hz1 => 1,
            Self::Hz1024 => 1024,
            Self::Hz4096 => 4096,
            Self::Hz8192 => 8192,
            Self::Hz32768 => 32768,
            Self::Custom(freq) => *freq,
        }
    }

    /// Create from Hz value
    pub fn from_hz(hz: u32) -> Self {
        match hz {
            1 => Self::Hz1,
            1024 => Self::Hz1024,
            4096 => Self::Hz4096,
            8192 => Self::Hz8192,
            32768 => Self::Hz32768,
            other => Self::Custom(other),
        }
    }
}

/// Square wave functionality trait
pub trait SquareWave: Rtc {
    /// Configure Frequency and enable square wave
    fn start_square_wave(&mut self, freq: SquareWaveFreq) -> Result<(), Self::Error>;

    /// Enable square wave output
    fn enable_square_wave(&mut self) -> Result<(), Self::Error>;

    /// Disable square wave output
    fn disable_square_wave(&mut self) -> Result<(), Self::Error>;

    /// Set the frequency (without enabling/disabling)
    fn set_square_wave_frequency(&mut self, freq: SquareWaveFreq) -> Result<(), Self::Error>;
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_to_hz_standard_frequencies() {
        assert_eq!(SquareWaveFreq::Hz1.to_hz(), 1);
        assert_eq!(SquareWaveFreq::Hz1024.to_hz(), 1024);
        assert_eq!(SquareWaveFreq::Hz4096.to_hz(), 4096);
        assert_eq!(SquareWaveFreq::Hz8192.to_hz(), 8192);
        assert_eq!(SquareWaveFreq::Hz32768.to_hz(), 32768);
    }

    #[test]
    fn test_to_hz_custom_frequencies() {
        assert_eq!(SquareWaveFreq::Custom(0).to_hz(), 0);
        assert_eq!(SquareWaveFreq::Custom(100).to_hz(), 100);
        assert_eq!(SquareWaveFreq::Custom(12345).to_hz(), 12345);
        assert_eq!(SquareWaveFreq::Custom(u32::MAX).to_hz(), u32::MAX);
    }

    #[test]
    fn test_from_hz_standard_frequencies() {
        assert_eq!(SquareWaveFreq::from_hz(1), SquareWaveFreq::Hz1);
        assert_eq!(SquareWaveFreq::from_hz(1024), SquareWaveFreq::Hz1024);
        assert_eq!(SquareWaveFreq::from_hz(4096), SquareWaveFreq::Hz4096);
        assert_eq!(SquareWaveFreq::from_hz(8192), SquareWaveFreq::Hz8192);
        assert_eq!(SquareWaveFreq::from_hz(32768), SquareWaveFreq::Hz32768);
    }

    #[test]
    fn test_from_hz_custom_frequencies() {
        assert_eq!(SquareWaveFreq::from_hz(0), SquareWaveFreq::Custom(0));
        assert_eq!(SquareWaveFreq::from_hz(50), SquareWaveFreq::Custom(50));
        assert_eq!(SquareWaveFreq::from_hz(2048), SquareWaveFreq::Custom(2048));
        assert_eq!(SquareWaveFreq::from_hz(9999), SquareWaveFreq::Custom(9999));
        assert_eq!(
            SquareWaveFreq::from_hz(u32::MAX),
            SquareWaveFreq::Custom(u32::MAX)
        );
    }

    #[test]
    fn test_round_trip_conversion() {
        let test_frequencies = vec![
            SquareWaveFreq::Hz1,
            SquareWaveFreq::Hz1024,
            SquareWaveFreq::Hz4096,
            SquareWaveFreq::Hz8192,
            SquareWaveFreq::Hz32768,
            SquareWaveFreq::Custom(0),
            SquareWaveFreq::Custom(42),
            SquareWaveFreq::Custom(2048),
            SquareWaveFreq::Custom(9876),
            SquareWaveFreq::Custom(u32::MAX),
        ];

        for original_freq in test_frequencies {
            let hz_value = original_freq.to_hz();
            let converted_back = SquareWaveFreq::from_hz(hz_value);
            assert_eq!(original_freq, converted_back);
        }
    }

    #[test]
    fn test_frequency_ordering() {
        let frequencies = [
            SquareWaveFreq::Hz1,
            SquareWaveFreq::Hz1024,
            SquareWaveFreq::Hz4096,
            SquareWaveFreq::Hz8192,
            SquareWaveFreq::Hz32768,
        ];

        let hz_values: Vec<u32> = frequencies.iter().map(|f| f.to_hz()).collect();

        for i in 1..hz_values.len() {
            assert!(hz_values[i] > hz_values[i - 1]);
        }
    }

    #[test]
    fn test_custom_frequency_edge_cases() {
        let edge_cases = vec![
            (0, SquareWaveFreq::Custom(0)),
            (2, SquareWaveFreq::Custom(2)),
            (1023, SquareWaveFreq::Custom(1023)),
            (1025, SquareWaveFreq::Custom(1025)),
            (4095, SquareWaveFreq::Custom(4095)),
            (4097, SquareWaveFreq::Custom(4097)),
            (8191, SquareWaveFreq::Custom(8191)),
            (8193, SquareWaveFreq::Custom(8193)),
            (32767, SquareWaveFreq::Custom(32767)),
            (32769, SquareWaveFreq::Custom(32769)),
        ];

        for (hz, expected) in edge_cases {
            assert_eq!(SquareWaveFreq::from_hz(hz), expected);
            assert_eq!(expected.to_hz(), hz);
        }
    }

    #[test]
    fn test_standard_frequencies_are_powers_of_two() {
        assert_eq!(SquareWaveFreq::Hz1024.to_hz(), 1024);
        assert_eq!(SquareWaveFreq::Hz4096.to_hz(), 4096);
        assert_eq!(SquareWaveFreq::Hz8192.to_hz(), 8192);
        assert_eq!(SquareWaveFreq::Hz32768.to_hz(), 32768);

        let freq_1024 = SquareWaveFreq::Hz1024.to_hz();
        let freq_4096 = SquareWaveFreq::Hz4096.to_hz();
        let freq_8192 = SquareWaveFreq::Hz8192.to_hz();
        let freq_32768 = SquareWaveFreq::Hz32768.to_hz();

        assert_eq!(freq_4096, freq_1024 * 4);
        assert_eq!(freq_8192, freq_4096 * 2);
        assert_eq!(freq_32768, freq_8192 * 4);
    }

    #[test]
    fn test_custom_with_standard_values() {
        let custom_1024 = SquareWaveFreq::Custom(1024);
        let custom_4096 = SquareWaveFreq::Custom(4096);

        assert_eq!(custom_1024.to_hz(), 1024);
        assert_eq!(custom_4096.to_hz(), 4096);

        assert_ne!(custom_1024, SquareWaveFreq::Hz1024);
        assert_ne!(custom_4096, SquareWaveFreq::Hz4096);
    }
}
```
