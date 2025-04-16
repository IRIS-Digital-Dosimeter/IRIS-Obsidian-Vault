
Welcome to the IRIS Notes! This site was made in Obsidian using the [Digital Garden extension](https://dg-docs.ole.dev/), hosted on [GitHub](https://github.com/Baron-Paelen/iris-notes-site), and deployed using [Vercel](https://vercel.com). Many of the pages here heavily paraphrase or quote the [SAMD51 Datasheet](https://ww1.microchip.com/downloads/aemDocuments/documents/MCU32/ProductDocuments/DataSheets/SAM-D5x-E5x-Family-Data-Sheet-DS60001507.pdf).

The purpose of this website is to hopefully demystify the software side of IRIS by presenting a more digestible version of the SAMD51's datasheet. As of the creation of this website, we've started migrating to the [[20 - Professional/21 - SCIPP/21.01 - IRIS/IRIS-Notes-Site/docs/M4 Board|M4 Board]], so most of the specifics will be tailored to the SAMD51/M4 board. 

In the sidebar, you will find all currently available notes. The *Code Analysis* folder contains notes on code lifted from external sources to help build intuition.
# Our Approach
## The Problem
The problem we're solving can be boiled down to orchestrating the following:
1. Generate readings from *4 input pins* via the ADC.
2. Store these readings in a comprehensible order/format on an SD card.
3. Collect the data at a frequency *over 1kHz*.

## The Breakdown
This is essentially a variation of the [Producer-Consumer Problem](https://en.wikipedia.org/wiki/Producer%E2%80%93consumer_problem). Obviously, there are things we must work around first due to hardware constraints:

3. Since the M0/M4 has a single core, reading from the ADC and writing to SD *cannot be done concurrently* by the CPU.
4. Since we have more input pins than ADCs, the readings will be *staggered*, causing long gaps between a pin's readings.
	- If using the ADC's averaging modes, these gaps will be even longer. Since the M4 has 2 ADCs, this problem is easier to handle, but requires more configuration.
5. Since SD cards write in 512 byte chunks and writes have noticeable latency, we must buffer the incoming data for efficiency's sake (N\*512 bytes).
6. Since the CPU's write to SD takes some time, then we must make sure the buffer always holds data that represents events across this time. Meaning, the data should be spaced out relative to the time it takes for the CPU to write one buffer to SD.

Based on the above, the workflow should look something like this:
```mermaid
graph LR
    Buffer[Buffer] --> CPU
    CPU[CPU] --> SD[SD Card]
    CPU --> DMAC[DMAC]
    DMAC --> CPU
    DMAC --> ADC[ADC]
    DMAC --> Buffer
    ADC --> DMAC
    A0[A0 - A4] --> ADC 


```
- **CPU**
	- Writes instructions and configurations to the DMAC
	- Writes the sample buffers to SD card when triggered
- **DMAC**
	- Flags/Triggers the CPU to begin writes when buffer is ready
	- Writes instructions and configurations to ADC via sequencing
- **ADC**
	- Flags DMAC once a result is ready
- **Buffer**
	- Holds ADC results in multiples of 512 bytes
- **Pins**
	- 4 input pins for the ADC
## M0 Approach
The original approach for collecting IRIS data used a [M0 Adalogger](https://www.adafruit.com/product/2796). It comes with a built in SD shield and the ability to use a small LiPo battery for independent operation. The main flaw with this board was the write latency - a single write would take about half a millisecond, which is far too long. 

Another flaw lay in the fact that the M0 only has 1 ADC, and the staggered readings between the 4 input pins introduced large gaps for each pin between SD writes. This means there are less datapoints between SD writes for each pin.

## M4 Approach
In order to allow concurrent data collection and SD writing, we use the M4's SAMD51's [[DMA controller]] to control the 2 onboard [[ADC|ADCs]]. The DMAC is configured to collect samples from the ADCs into a buffer. Once the buffer is filled, the CPU begins the process of writing the buffer's contents into the SD card. The DMAC then starts new conversions on the ADCs.

### Buffers
Each ADC will get its own result buffer. Each half of a buffer will take turns being written to. This means that any one time, one half will be *volatile*, being operated on by the DMAC, while the other half will be ready for operations, like writing to SD.

***We may switch to one, single buffer containing all results later.*** This may help during the SD write process.
### ADCs
The current plan is for each ADC to be given 2 pins each to alternate between. This lessens the gaps between each pin's samples. Using *DMA Sequencing*, each ADC will be controlled via their own sequencing descriptor which will be responsible for updating the ADC's next input. These descriptors will constantly point back to themselves, allowing us to repeat the input switching endlessly. We will refer to these descriptors as *input descriptors.*

Each ADC will also get two descriptors to handle collecting the ready conversions from the result registers (four total output descriptors). The first descriptor is responsible for entering results into the first half of the respective buffer, while the second descriptor is responsible for entering results into the second half of the respective buffer. These descriptors will be referred to as *output descriptors*.

***We may switch to one, single buffer containing all results later.*** This may help during the SD write process. This can be done by changing the `dstaddr` field for the *output descriptors*.

### SD
TODO. #Stub 

### Timestamps
TODO. #Stub 

### Places that need work or investigation?
- Should our buffer(s) for the ADC be 512 bytes or multiples of 512 bytes? Will the SD card overhead get in the way of bigger buffers?
- Should we have one buffer and interleave everything into it instead of the two right now?
- What's the best way to get timestamps for each reading or set of readings? Get a `micros()` call every time a DMA transfer is complete and do slight math? 
- **Best way to make sure timing aligns?** 
	- **t1:** time to fill the buffer(s) w/ staggered samples (possibly averaged) to be taken and dropped into buffer. 
	- **t2:** time for SD to write the buffer and finish
	- **t3:** time to add timings to the buffers, or create a new buffer that holds structs which match the current `serial_binary` format.
	- t1+t3 must be slightly less than t2, but the buffer must hold evenly spaced data, or damn near it. If the data isn't evenly spaced, then getting accurate timing for each sample will be difficult.