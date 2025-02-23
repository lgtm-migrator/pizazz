# Pizazz

> A utility class to leverage 74HC595 shift register chips with a Raspberry Pi.

[![PyPI version](https://badge.fury.io/py/pizazz.svg)](https://badge.fury.io/py/pizazz)
[![Documentation Status](https://readthedocs.org/projects/pizazz/badge/?version=latest)](https://pizazz.readthedocs.io/en/latest/?badge=latest)
[![Downloads](https://static.pepy.tech/personalized-badge/pizazz?period=total&units=international_system&left_color=black&right_color=orange&left_text=Downloads)](https://pepy.tech/project/pizazz)
[![pre-commit][pre-commit-image]][pre-commit-url]
[![Imports: isort][isort-image]][isort-url]
[![Code style: black][black-image]][black-url]
[![Checked with mypy][mypy-image]][mypy-url]
[![security: bandit][bandit-image]][bandit-url]
[![licence: mit][mit-license-image]][mit-license-url]

![](https://github.com/Stephen-RA-King/pizazz/raw/main/assets/header.png)

The 74HC595 shift register is an incredibly useful chip. A single chip has 8 output pins which can be
controlled with only 3 input pins (excluding Vcc and Gnd of course).
That is great in itself however 595's can be daisy-chained together to give you multiples of 8 pin outputs yet still
always controlled by only 3 input pins! Wow!

If you are not sure why this is useful then let me explain.

I had a requirement to create a LED "Status Board" for a monitoring and automation application that I am also writing.
The status board would reflect the current operation status of things like Jenkins jobs, Github Actions, Linux services etc etc.
I needed a minimum of 16 LEDs. Now there already exists a [**status board**][status-board-url] HAT. However it only tracks 5 items.

Using the Raspberry [**RPi.GPIO**][rpi-gpio-url] library it is possible to individually switch the 27 GPIO pins. However each LED would require
a wire from the GPIO pin. This is very physically unwieldy and clunky to control in python.

Enter the 74HC595...

This class enables you to individually control any number of LEDS (or other output devices) with only 3 GPIO pins.

### Basic Wiring of the 74HC595 8-bit shift register to a Raspberry Pi

![](https://github.com/Stephen-RA-King/pizazz/raw/main/assets/1chip.png)

| Pin   | Tag     | Description                   |
| ----- | ------- | ----------------------------- |
| 1 - 7 | Q1 - Q7 | Parallel Data output pins 1-7 |
| 8     | Gnd     | Ground                        |
| 9     | Q7->    | Serial data output pin        |
| 10    | MR      | Master Reset                  |
| 11    | SH      | Clock pin                     |
| 12    | ST      | Latch pin                     |
| 13    | OE      | Output enable                 |
| 14    | DS      | Serial data input             |
| 15    | Q0      | Parallel data output pin 0    |
| 16    | Vcc     | Positive voltage supply       |

### Chaining 2 or more shift registers together

![](https://github.com/Stephen-RA-King/pizazz/raw/main/assets/2chip.png)

### How the register works

The 595 has two registers (which can be thought of as “memory containers”), each with just 8 bits of data.

1. The Shift Register
2. The Storage Register

Whenever we apply a clock pulse to a 595, two things happen:

1. The bits in the Shift Register move one step to the left. For example, Bit 7 accepts the value that was previously in bit 6, bit 6 gets the value of bit 5 etc.

2. Bit 0 in the Shift Register accepts the current value on DATA pin. At the rising edge of the pulse, if the data pin is high, then a 1 gets pushed into the shift register. Otherwise, it is a 0.

On enabling the Latch pin, the contents of Shift Register are copied into the Storage Register.
Each bit of the Storage Register is connected to one of the output pins Q0–Q7 of the IC, so in general, when the value in the Storage Register changes, so do the outputs.

## Installation

Raspberry Pi:

```sh
pip3 install pizazz
```

## Connecting the Raspberry Pi

The 40 pins of the Raspberry Pi are GPIO, 5v, 3.3V and ground. Some of the GPIO pins can have special purposes.
However, all of them can be controlled by the RPi.GPIO Python Library.
The RPi.GPIO requires that you specify how you will identify the pins that you use. There are 2 ways:

1. **GPIO.BOARD:** option specifies that you are referring to the pins by the number of the pin.

2. **BCM:** option means that you are referring to the pins by the "Broadcom SOC channel" number, these are
   the numbers after "GPIO"

So referring to the diagram below: BCM mode GPIO2 is the same as BOARD mode pin 2

![](https://github.com/Stephen-RA-King/pizazz/raw/main/assets/40pinheader.png)

Connect any GPIO's to the clock, latch and data pins of the register and connect the the 5v supply and earth
as indicated in the register diagram.

## Library Usage examples

_For more examples and usage, please refer to the [Wiki][wiki]._

https://user-images.githubusercontent.com/33905365/163853342-a1a19d5e-e392-483d-845f-40bfe25d4cb3.mp4

Import the library

```sh
from pizazz.pizazz import HC595
```

Instantiate the library passing the initialisation arguments

```sh
shifter = HC595(mode="BCM", data=17, clock=27, latch=18, ics=2)
```

the 'ics' parameter defines the number of registers daisey-chained together.

There are four public methods in the library:

| Method        | Description                              |
| ------------- | ---------------------------------------- |
| clear()       | sets shift and storage registers to zero |
| test()        | Cycles sequentially through all outputs  |
| set_output()  | explicitly sets specific pin outputs     |
| set_pattern() | sets output using a bit pattern          |

### 1. Using the set_output(output, mask) method

Args:

**output** (int) - decimal value of the binary bits you want to set to "on"

**mask** (int) - decimal value of the binary bits to consider in this operation.

Consider the following setup:

![](https://github.com/Stephen-RA-King/pizazz/raw/main/assets/masking.png)

Using a mask has 2 benefits:

1. It enables the library to explicitly turn LEDS 'off'. e.g. sending an output value of 16 means turn pin 5 'on'. it has no concept of turning pin 6 'off'. Using a mask facilitates this.
2. It isolates the pins to consider in the update. For a status board this is important. The inputs from the sensors can now be considered in isolation from the other sensors making asynchronous updates possible.

Consider sensor 2:

| method values      | LED3 | LED4 |
| ------------------ | ---- | ---- |
| set_output(0, 12)  | OFF  | OFF  |
| set_output(4, 12)  | ON   | OFF  |
| set_output(8, 12)  | OFF  | ON   |
| set_output(12, 12) | ON   | ON   |

**NOTE: All other LED outputs remain the same and are untouched by these operations**

This now makes programming the shift register a simple process. e.g. consider a Jenkins job

```sh
jenkins_mask = 48
jenkins_pass = 16
jenkins_fail = 32

# 'sensor' receives a failing indication
shifter.set_output(jenkins_fail, jenkins_mask)

# 'sensor' receives a passing indication
shifter.set_output(jenkins_pass, jenkins_mask)

```

The second value is the bit mask (similar to an IP bit mask) - Explained later

### 2. Using the set_pattern(chip_pattern) method

Args:

**chip_pattern** (List or Nested List) - Bit pattern representing the pins to set 'on' or 'off'. If more than two registers are used then the pattern should be a nested list.

Using the bit pattern (for a two chip configuration)

```sh
shifter.set_pattern([[0, 0, 1, 1, 0, 0, 0, 0], [0, 0, 0, 0, 0, 0, 0, 0]])
```

For a single chip a simple list should be used:

```sh
shifter.set_pattern([0, 0, 1, 1, 0, 0, 0, 0])
```

## Documentation

[**Read the Docs**](https://pizazz.readthedocs.io/en/latest/)

## Meta

[![](https://github.com/Stephen-RA-King/pizazz/raw/main/assets/linkedin.png)](https://linkedin.com/in/stephen-k-3a4644210)
[![](https://github.com/Stephen-RA-King/pizazz/raw/main/assets/github.png)](https://github.com/Stephen-RA-King/Stephen-RA-King)
[![](assets/pypi.png)](https://pypi.org/project/pizazz/)
[![](https://github.com/Stephen-RA-King/pizazz/raw/main/assets/www.png)](https://www.justpython.tech)
[![](https://github.com/Stephen-RA-King/pizazz/raw/main/assets/email.png)](mailto:stephen.ra.king@gmail.com)

Author: Stephen R A King

Distributed under the MIT License. See `LICENSE` for more information.

<!-- Markdown link & img dfn's -->

[rpi-gpio-url]: https://pypi.org/project/RPi.GPIO/
[status-board-url]: https://thepihut.com/products/status-board-pro
[pre-commit-image]: https://img.shields.io/badge/pre--commit-enabled-brightgreen?logo=pre-commit&logoColor=white
[pre-commit-url]: https://github.com/pre-commit/pre-commit
[isort-image]: https://img.shields.io/badge/%20imports-isort-%231674b1?style=flat&labelColor=ef8336
[isort-url]: https://pycqa.github.io/isort/
[black-image]: https://img.shields.io/badge/code%20style-black-000000.svg
[black-url]: https://github.com/psf/black
[bandit-image]: https://img.shields.io/badge/security-bandit-yellow.svg
[bandit-url]: https://github.com/PyCQA/bandit
[mypy-image]: http://www.mypy-lang.org/static/mypy_badge.svg
[mypy-url]: http://mypy-lang.org/
[mit-license-image]: https://img.shields.io/badge/license-MIT-blue
[mit-license-url]: https://choosealicense.com/licenses/mit/
[wiki]: https://github.com/stephen-ra-king/pizazz/wiki
