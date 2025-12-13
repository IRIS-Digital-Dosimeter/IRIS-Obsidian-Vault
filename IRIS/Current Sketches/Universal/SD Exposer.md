---
author: Andrew Y
---
## Requirements
- M4 **or** M0 board
- Adalogger featherwing
## Overview
This sketch exposes the onboard SD card as an external storage device to whatever computer the microcontroller is hooked up to.

Slightly modified from the Arduino example `msc_sdfat.ino`.

> [!NOTE] 
> - Follows standard `config.h` format.
> - Requires the `TinyUSB` USB stack.
## Config Values

| **Entry**     | **Note**                                                 |
| ------------- | -------------------------------------------------------- |
| SD_CS_PIN     | Chip Select pin. **10** for M4 and **4** for M0          |
| SPI_CLOCK_MHZ | SPI clock speed. **50** max for M4 and **12** max for M0 |
| SPI_CLOCK     | Leave as-is. Not configurable.                           |
| SD_CONFIG     | Leave as-is. Not configurable.                           |
