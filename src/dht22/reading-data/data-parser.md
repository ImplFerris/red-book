# Data Parser

The parse_data function takes the raw data bytes from the sensor and converts them into meaningful temperature and humidity values. The DHT22 sends two bytes for humidity and two for temperature.

```rust
fn parse_data(&self, data: [u8; 4]) -> Reading {
    let [hum_hi, hum_lo, temp_hi, temp_lo] = data;

    let joined_humidity = u16::from_be_bytes([hum_hi, hum_lo]);
    let relative_humidity = joined_humidity as f32 / 10.0;

    let is_temp_negative = (temp_hi >> 7) != 0;
    let temp_hi = temp_hi & 0b0111_1111;
    let joined_temp = u16::from_be_bytes([temp_hi, temp_lo]);
    let mut temperature = joined_temp as f32 / 10.0;
    if is_temp_negative {
        temperature = -temperature;
    }

    Reading {
        temperature,
        relative_humidity,
    }
}
```
## Decode Humidity

The humidity is received as two bytes: hum_hi and hum_lo. These two bytes together form a 16-bit number. We join them using:
```rust
let joined_humidity = u16::from_be_bytes([hum_hi, hum_lo]);
```

The actual humidity is this value divided by 10.0:
```rust
let relative_humidity = joined_humidity as f32 / 10.0;
```
For example, if the sensor sends 0x02 and 0x58 (which is 600 in decimal), the humidity will be 600 / 10 = 60.0%.


## Decode Temperature
The temperature is also received as two bytes: temp_hi and temp_lo.

However, the sensor uses the highest bit of "temp_hi" to indicate if the temperature is negative. So first, we check if that bit is set:
```rust
let is_temp_negative = (temp_hi >> 7) != 0;
```

We then clear the sign bit (bit 7) before combining the bytes:

```rust
let temp_hi = temp_hi & 0b0111_1111;
let joined_temp = u16::from_be_bytes([temp_hi, temp_lo]);
let mut temperature = joined_temp as f32 / 10.0;
```

If the temperature was marked as negative, we flip the sign:

```rust
 if is_temp_negative {
    temperature = -temperature;
}
```
Finally, we return the result:
```rust
 Reading {
    temperature,
    relative_humidity,
}
```

## Testing parser

There is nothing complex going on in these tests. We are simply checking if the parse_data function correctly converts raw bytes into meaningful temperature and humidity values. 

In the first test, we pass a sample input representing 55.5% humidity and 24.6°C temperature and assert that the output matches. 

In the second test, we use a special case where the temperature is negative (-1.0°C). The most significant bit (bit 7) in the temperature high byte is set, which signals a negative temperature.

```rust
#[test]
fn test_parse_data_positive_temp() {
    let mut pin = PinMock::new(&[]);

    let dht = Dht22::new(pin.clone(), NoopDelay);
    // Humidity: 55.5% -> [0x02, 0x2B] => 555
    // Temperature: 24.6C -> [0x00, 0xF6] => 246
    let data = [0x02, 0x2B, 0x00, 0xF6];

    let reading = dht.parse_data(data);

    assert_eq!(
        reading,
        Reading {
            relative_humidity: 55.5,
            temperature: 24.6,
        }
    );
    pin.done();
}

#[test]
fn test_parse_data_negative_temp() {
    let mut pin = PinMock::new(&[]);

    let dht = Dht22::new(pin.clone(), NoopDelay);

    // Humidity: 40.0% -> [0x01, 0x90] => 400
    // Temperature: -1.0C -> [0x80, 0x0A]
    // Bit 7 of temp_hi is 1 => negative
    // Clear sign bit: 0x80 & 0x7F = 0x00, so [0x00, 0x0A] = 10 => 1.0 then negated
    let data = [0x01, 0x90, 0x80, 0x0A];

    let reading = dht.parse_data(data);

    assert_eq!(
        reading,
        Reading {
            relative_humidity: 40.0,
            temperature: -1.0,
        }
    );
    pin.done();
}
```
