
# Details of Time Data Bits

Now let's look at how the data is stored inside each register. Remember, each register holds 8 bits (bit 7 to bit 0), and the time values are stored in BCD format.
<style>
  .styled-table {
    width: 100%;
    border-collapse: collapse;
    font-family: "Roboto", Arial, sans-serif;
    font-size: 14px;
    text-align: center;
    border-radius: 6px;
    overflow: hidden;
    box-shadow: 0 2px 6px rgba(0,0,0,0.15);
  }

  .styled-table th {
    background-color: #1c1c1c;
    color: #ffffff;
    padding: 10px;
    font-weight: 600;
    border: 1px solid #424242;
  }

  .styled-table td {
    padding: 10px;
    border: 1px solid #424242;
  }

  .styled-tr:nth-child(even) {
    background-color: #2e2e2e;
  }

  .register-name {
    background-color: #3a3a3a;
    color: #ffffff;
    font-weight: 600;
    text-align: left;
    padding-left: 12px;
  }

  .special-bit {
    background-color: #bf360c;
    color: #ffffff;
    font-weight: 500;
  }

  .fixed-bit {
    background-color: #546e7a;
    color: #ffffff;
  }

  .group-tens {
    background-color: #1565c0;
    color: #ffffff;
    font-weight: 500;
  }

  .group-ones {
    background-color: #2e7d32;
    color: #ffffff;
    font-weight: 500;
  }
</style>

<table class="styled-table">
    <tr class="styled-tr">
        <th>REGISTER</th>
        <th>BIT 7</th>
        <th>BIT 6</th>
        <th>BIT 5</th>
        <th>BIT 4</th>
        <th>BIT 3</th>
        <th>BIT 2</th>
        <th>BIT 1</th>
        <th>BIT 0</th>
    </tr>
    <tr>
        <td class="register-name">Seconds</td>
        <td class="special-bit">CH</td>
        <td colspan="3" class="group-tens">Tens digit of seconds</td>
        <td colspan="4" class="group-ones">Ones digit of seconds</td>
    </tr>
    <tr>
        <td class="register-name">Minutes</td>
        <td class="fixed-bit">0</td>
        <td colspan="3" class="group-tens">Tens digit of minutes</td>
        <td colspan="4" class="group-ones">Ones digit of minutes</td>
    </tr>
    <tr>
        <td class="register-name">Hours</td>
        <td class="fixed-bit">0</td>
        <td class="special-bit">Select <br/>12/24 format</td>
        <td class="special-bit">PM/AM Flag in 12 hour format<br/>or<br/> Part of Tens in 24 hour format</td>
        <td class="group-tens">Tens digit of hours</td>
        <td colspan="4" class="group-ones">Ones digit of hours</td>
    </tr>
    <tr>
        <td class="register-name">Day</td>
        <td class="fixed-bit">0</td>
        <td class="fixed-bit">0</td>
        <td class="fixed-bit">0</td>
        <td class="fixed-bit">0</td>
        <td class="fixed-bit">0</td>
        <td colspan="3" class="group-ones">Day value (1-7)</td>
    </tr>
    <tr>
        <td class="register-name">Date</td>
        <td class="fixed-bit">0</td>
        <td class="fixed-bit">0</td>
        <td colspan="2" class="group-tens">Tens digit of date</td>
        <td colspan="4" class="group-ones">Ones digit of date</td>
    </tr>
    <tr>
        <td class="register-name">Month</td>
        <td class="fixed-bit">0</td>
        <td class="fixed-bit">0</td>
        <td class="fixed-bit">0</td>
        <td class="group-tens">Tens digit of month</td>
        <td colspan="4" class="group-ones">Ones digit of month</td>
    </tr>
    <tr>
        <td class="register-name">Year</td>
        <td colspan="4" class="group-tens">Tens digit of year</td>
        <td colspan="4" class="group-ones">Ones digit of year</td>
    </tr>   
</table>

## Seconds: Range 00-59

The bits 0-3 represent the ones digit part of the seconds. The bits 4-6 represent the tens digit part of the seconds.

If you want to represent seconds 30, the BCD format will be: 0011 0000. So, we will put 011 in bits 4-6 (tens digit) and 0000 in bits 0-3 (ones digit).

```sh
Bit:     7 6 5 4 3 2 1 0
Value:   0 0 1 1 0 0 0 0
         │ └─┬─┘ └──┬──┘
         │   3      0
         CH bit
```

CH bit (bit 7) controls clock halt - it's like an on/off switch for the entire clock (1 = clock stopped, 0 = clock running).

## Minutes: Range 00-59

The bits 0-3 represent the ones digit part of the minutes. The bits 4-6 represent the tens digit part of the minutes.

If you want to represent minutes 45, the BCD format will be: 0100 0101. So, we will put 100 in bits 4-6 (tens digit) and 0101 in bits 0-3 (ones digit).

```sh
Bit:     7 6 5 4 3 2 1 0
Value:   0 1 0 0 0 1 0 1
         │ └─┬─┘ └──┬──┘
         │   4      5
         Always 0
```

Bit 7 is always 0.

## Hours: Range 1-12 + AM/PM (12-hour mode) or 00-23 (24-hour mode)

In 12-hour mode:
- Bit 6 = 1 (selects 12-hour format)
- Bit 5 = AM/PM bit (0=AM, 1=PM)
- Bit 4 = tens digit of hour (0 or 1)
- Bits 3-0 = ones digit of hour

In 24-hour mode:
- Bit 6 = 0 (selects 24-hour format)
- Bits 5-4 = tens digit of hour
- Bits 3-0 = ones digit of hour

> Important: The hours value must be re-entered whenever the 12/24-hour mode bit is changed.

If you want to represent 2 PM in 12-hour mode, the format will be: 0110 0010. So bit 6=1 (12-hour), bit 5=1 (PM), bit 4=0 (tens digit), bits 3-0 will be 0010 (ones digit). 

```sh
Bit:     7 6 5 4 3 2 1 0
Value:   0 1 1 0 0 0 1 0
         │ │ │ │ └──┬──┘
         │ │ │ │    2
         │ │ │ tens digit
         │ │ PM bit
         │ 12-hour format
         Always 0
```

If you want to represent 14:00 in 24-hour mode, the format will be: 0001 0100. So bit 6=0 (24-hour), bits 5-4 will be 01 (tens digit), bits 3-0 will be 0100 (ones digit).

```sh
Bit:     7 6 5 4 3 2 1 0
Value:   0 0 0 1 0 1 0 0
         │ │ └┬┘ └──┬──┘
         │ │  1     4
         │ 24-hour format
         Always 0
```

## Day: Range 01-07 (1=Sunday, 2=Monday, etc.)

The bits 0-2 represent the day value. Bits 3-7 are always 0.

If you want to represent Tuesday (day 3), the format will be: 0000 0011.

```sh
Bit:     7 6 5 4 3 2 1 0
Value:   0 0 0 0 0 0 1 1
         └───┬───┘ └─┬─┘
         Always 0     3
```

## Date: Range 01-31 (day of month)

The bits 0-3 represent the ones digit part of the date. The bits 4-5 represent the tens digit part of the date.

If you want to represent date 25, the BCD format will be: 0010 0101. So, we will put 10 in bits 4-5 (tens digit) and 0101 in bits 0-3 (ones digit).

```sh
Bit:     7 6 5 4 3 2 1 0
Value:   0 0 1 0 0 1 0 1
         └┬┘ └┬┘ └──┬──┘
          0   2     5
         Always 0
```

Bits 6-7 are always 0.

## Month: Range 01-12

The bits 0-3 represent the ones digit part of the month. Bit 4 represents the tens digit part of the month.

If you want to represent month 12 (December), the BCD format will be: 0001 0010. So, we will put 1 in bit 4 (tens digit) and 0010 in bits 0-3 (ones digit).

```sh
Bit:     7 6 5 4 3 2 1 0
Value:   0 0 0 1 0 0 1 0
         └─┬─┘ │ └──┬──┘
     Always 0  1    2
```

Bits 5-7 are always 0.

## Year: Range 00-99 (represents 2000-2099)

The bits 0-3 represent the ones digit part of the year. The bits 4-7 represent the tens digit part of the year.

If you want to represent year 23 (2023), the BCD format will be: 0010 0011. So, we will put 0010 in bits 4-7 (tens digit) and 0011 in bits 0-3 (ones digit).

```sh
Bit:     7 6 5 4 3 2 1 0
Value:   0 0 1 0 0 0 1 1
         └──┬──┘ └──┬──┘
            2      3
```

Remember when I first mentioned the DS1307 and how it could only represent dates until 2100? I was curious about that limitation too. Well, Now we've identified the culprit: the DS1307's 8-bit year register can only store two digits (00-99) in BCD format, and the chip assumes these always represent years in the 2000s century.
