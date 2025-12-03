---
author: Andrew Y
---
## Requirements
- M4 board
- Adalogger featherwing
- SD card (exFat or FAT32)
## Overview
This sketch is the successor of previous M0 attempts at a proper lossless data logging tool. It utilizes the DMA controller to automatically switch between input pins and to place ADC results from result registers into a buffer. It also generates timestamps at regular intervals defined by `NUM_RESULTS`. Currently, configured to log at ~1KHz.

> [!NOTE] 
> Follows standard `config.h` format.

## Config Values
#TODO