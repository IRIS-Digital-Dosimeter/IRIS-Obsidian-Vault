---
author: Andrew Y
---
## Requirements
- M4 board
- Adalogger Featherwing
- SD card (exFat or FAT32)
## Overview
This sketch is the successor of previous M0 attempts at a proper lossless data logging tool. It utilizes the DMA controller to automatically switch between input pins and to place ADC results from result registers into a buffer. It also generates timestamps at regular intervals defined by `NUM_RESULTS`. Currently, configured to log at ~1KHz.

> [!NOTE] 
> Follows standard `config.h` format.

## Config Values
#### File Allocation Configuration

| **Entry**       | **Note**                                                                                     |
| --------------- | -------------------------------------------------------------------------------------------- |
| BYTES_PER_VALUE | Leave as-is. Not configurable.                                                               |
| VALUES_PER_LINE | Leave as-is. Not configurable.                                                               |
| BUF_SAMPLES     | Should be a power of 2. The bigger this value, the more samples are written to SD at a time. |
| SHIFT_MULT      | Number of left bit-shifts to apply when calculating file preallocation size                  |
#### SD Configuration 

| **Entry**         | **Note**                                                 |
| ----------------- | -------------------------------------------------------- |
| SD_CS_PIN         | Chip Select pin. **10** for M4 and **4** for M0          |
| SPI_CLOCK_MHZ     | SPI clock speed. **50** max for M4 and **12** max for M0 |
| WRITE_BUFFER_SIZE | Leave as-is. Not configurable.                           |
| SPI_CLOCK         | Leave as-is. Not configurable.                           |
| SD_CONFIG         | Leave as-is. Not configurable.                           |
#### ADC Configuration 

| **Entry**             | **Note**                                                                                                                                                 |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| USE_AVG_MODE          | Toggles averaging mode on the ADCs. **Currently non-functioning - leave at 0**                                                                           |
| NUM_RESULTS           | Number of ADC samples to be buffered before being aligned into SD write buffer. Each ADC gets one of these. The max that can fit in M4's RAM is ~11,000. |
| ADC_SAMPLEN           | Number of ADC cycles per sample.                                                                                                                         |
| ADC_FACTOR_VAL        | ADC's prescaler division value. Must be a power of 2 up to 256. Divide's GCLK clock speed by `ADC_FACTOR_VAL`.                                           |
| ADC_PRESCALING_FACTOR | Leave as-is. Not configurable.                                                                                                                           |
#### GCLK Configuration 

| **Entry**       | **Note**                                                                                                                                                      |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| GCLK_DIV_FACTOR | Generic Clock generator's prescaler division value. Divides GCLK clock speed by `GCLK_DIV_FACTOR`. (This is not a power of 2 like the ADC prescaling factor). |
| ADC_GCLK        | The ID of the generic clock generator to use. Use `2` as it's unused.                                                                                         |
#### ADC Input Pins 
See the mapping [here](https://cdn-learn.adafruit.com/assets/assets/000/111/181/original/arduino_compatibles_Adafruit_Feather_M4_Express.png?1651092186).

| **Entry**    | **Note**                |
| ------------ | ----------------------- |
| ADC0_INPUT_0 | ADC0's first input pin  |
| ADC0_INPUT_1 | ADC0's second input pin |
| ADC1_INPUT_0 | ADC1's first input pin  |
| ADC1_INPUT_1 | ADC1's second input pin |

#### Debug Print Configuration 

| **Entry**   | **Note**                                                   |
| ----------- | ---------------------------------------------------------- |
| DEBUG_R0_P0 | Enable or disable debug prints for ADC0's first input pin  |
| DEBUG_R0_P1 | Enable or disable debug prints for ADC0's second input pin |
| DEBUG_R1_P0 | Enable or disable debug prints for ADC1's first input pin  |
| DEBUG_R1_P1 | Enable or disable debug prints for ADC1's second input pin |
