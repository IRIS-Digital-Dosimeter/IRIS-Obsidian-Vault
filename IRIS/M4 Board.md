---
author: Andrew Y

---

Currently, the IRIS project is transitioning from the [Adafruit M0 Adalogger](https://www.adafruit.com/product/2796) to the [M4 Express](https://www.adafruit.com/product/3857) and [Datalogging FeatherWing](https://www.adafruit.com/product/2922). The transition is being made because the SD write latency with the M0 was simply too high, and the M0's singular ADC proved restrictive for higher frequency sampling. The M4's SAMD51 chip has two [[ADC|ADCs]], and a significantly higher clock speed. This means we can increase our logging and sampling frequencies significantly.

Additionally, The SAMD51's [[DMA Controller]] supports *DMA Sequencing*, which allows the DMAC to automatically update registers of both [[ADC|ADCs]]  via DMA descriptors. This allows us to switch which pins each ADC is hooked up to automatically.

As described in the [[20 - Professional/21 - SCIPP/21.01 - IRIS/IRIS/IRIS|IRIS]] notes, the current approach is as follows:

```mermaid
sequenceDiagram
	participant ADC1
	participant ADC2
	participant DMAC
	participant CPU
	
	participant circ as Circular Buffer
	participant SD

	CPU-->>CPU: Sketch starts

	par Configurations set via DMA descriptors
		CPU->>ADC1: 
	and 
		CPU->>ADC2: 
	and
		CPU->>DMAC: 
	end

	loop For Writing to SD
		loop For Filling Buffer
			loop For Each Sample
				par Swap sampling pins
					DMAC->>ADC1: 
				and	
					DMAC->>ADC2: 
				end
				
				par ADCs sample
					ADC1-->>DMAC: RESRDY
					DMAC->>circ: Store ADC1 result
				and
					ADC2-->>DMAC: RESRDY
					DMAC->>circ: Store ADC2 result
				end
			end	
		end
		
		Note over ADC1,circ: Buffer is full
		CPU->>SD: Write Buffer contents to SD
		Note over ADC1,circ: Buffer now safe to overwrite
	end
			


```

In order to allow concurrent data collection and SD writing, we use the M4's [[DMA controller]] to address the 2 onboard [[ADC|ADCs]]. The DMAC is configured to collect samples from the ADCs into a buffer. Once the buffer is filled, the CPU begins the process of writing the buffer's contents into the SD card. The DMAC then starts new conversions on the ADCs.

The 2 ADCs each alternate between 2 pins, interleaving the results into the buffer (A0, A1, A2, A3, A1, A2, etc).

> [!warning]
> Looks like Adafruit's SPI library overwrites register level DMAC programming according to [[Resources#^99fd12|MartinL]], which may be causing the issues that came up with using the M0's SD card with DMA. 
> 
> Solution found - **see relevant section in [[SdFat Library]].**