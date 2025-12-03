---
author: Andrew Y
---
## Requirements
- M4 board
- Adalogger featherwing
- Toggle switch
- SD card (exFat or FAT32)
## Overview
This sketch is a custom version of the [[M4 Datalogger]] sketch meant for the 2025 MD Anderson trip. The main difference is that this sketch *only records data when a toggle switch is flipped on*. This better suits the use case of MDA personnel. It's currently operating at a ~11KHz collection rate.

> [!NOTE] 
> Follows standard `config.h` format.

## Config Values
#TODO