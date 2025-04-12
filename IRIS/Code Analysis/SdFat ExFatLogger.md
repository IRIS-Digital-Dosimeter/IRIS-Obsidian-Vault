**Source:** [SdFat/examples/ExFatLogger at master · adafruit/SdFat · GitHub](https://github.com/adafruit/SdFat/tree/master/examples/ExFatLogger)

This is a pretty feature-rich logger, though it likely has everything we need in terms of capturing data to file. They use analogRead(), which we won't be doing, instead opting to read off of our DMA-filled buffers. There is also a menu of sorts that we will also be ignoring. 

Functions and globals seem to be quite entangled, which makes it a bit difficult to follow, especially as I'm trying to move helper functions into a separate header.

## Areas of Interest
### `logData()`
- contains the logic for actually writing binary data to card
- uses a FIFO buffer system
	- likely able to skip this entirely as we already have data coming in as buffers
- it's also just a big `while(true)`
- 
### `printData()`
- reads data out of the current binary file
- probably wont need, but *good example of entanglement*
### `setup()`