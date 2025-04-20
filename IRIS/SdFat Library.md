---
author: Andrew Y

---

> [!tldr]-  Reference
> [[Resources#^9eb02b|Arduino Reference]]
> [[Resources#^1259ff|Source Repo]]
> 

We will be switching to the *SdFat* library over the standard *SD* library for performance reasons. The standard *SD* library is a wrapper for *SdFat*, which introduces a lot of performance overhead.

**This is not an exhaustive guide. These are simply notes that can help you avoid the troubleshooting, experimentation, and profiling that led to all this info.**

# General Notes
- M0's max SPI clock frequency is 12MHz
- M4's max SPI clock frequency is 50MHz
	- they are pretty adamant that it cannot go higher even if you overclock the SAMD51
- **M4's chip select pin is 10**
- **SD cards are addressed in 512 byte sectors**
	- keep our writes and files in multiples of 512 for best performance and least fragmentation
## Files
- Try to align file size to allocation unit size for fastest operations 
- **Preallocation is Key!**
	- *Preallocation* dedicates a contiguous block of flash to your file. This makes writes and reads much, *much* faster.
	- `file.preAllocate(u64)`
		- takes number of bytes as argument
# Configuration
## General Config
1. Define your desired *SPI configuration*. Use `SdSpiConfig(CS pin, DEDICATED_SPI, SD_SCK_MHZ(ulong))`. This includes:
	- **SD chip select pin**
	- **SPI mode**
		- Dedicated (preferred)
			- SD card is only device on SPI bus
			- `DEDICATED_SPI`
		- Shared
			- SPI bus is shared with other devices
			- `SHARED_SPI`
	- **SPI Clock frequency**
		- M4 should be 50Mhz
		- M0 should be 12Mhz (found through experimentation)
### Example
```cpp
const uint8_t SD_CS_PIN = 10;
#define SPI_CLOCK SD_SCK_MHZ(50)
#define SD_CONFIG SdSpiConfig(SD_CS_PIN, DEDICATED_SPI, SPI_CLOCK)

sd.begin(SD_CONFIG);
```

## When Using in Conjunction with DMAC Code
Calling `sd.begin()` automatically configures the DMAC's descriptor section, so when initializing DMA descriptors and whatnot, add the following to point your descriptor and writeback sections to those created by *SdFat*:
```cpp
descriptor_section =  reinterpret_cast<dmadesc(*)[DMAC_CH_NUM]>(DMAC->BASEADDR.reg); //point array pointer to BASEADDR defined by SD.begin

wrb =  reinterpret_cast<dmadesc(*)[DMAC_CH_NUM]>(DMAC->WRBADDR.reg);                 //point array pointer to WRBADDR defined by SD.begin

DMAC->CTRL.reg = DMAC_CTRL_DMAENABLE | DMAC_CTRL_LVLEN(0xf);
```

