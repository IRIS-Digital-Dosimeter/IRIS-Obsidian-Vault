
**Source:** [SAMD51 ADC0 and ADC1 Multiplexing with DMA - Hardware / Arduino Zero - Arduino Forum](https://forum.arduino.cc/t/samd51-adc0-and-adc1-multiplexing-with-dma/1238002/8)

**This one's interesting because raidersnake not only had a very similar objective, but also solved the issue of programming the [[DMA Controller|DMAC]] and using SPI/SD card in the same sketch without locking up the board.** **However, it came with some errors that stop ADC1 from functioning continuously without some tweaks. Even then, ADC1 does not seem to be giving real results.**

**As of writing this, a modified version exists in the sandbox_DMA branch as `dma_ADC_linked_descriptors` - [commit 8d065dcaa65aab43ac917459d1054ec46d87ec81](https://github.com/IRIS-Digital-Dosimeter/IRIS-Project/commit/8d065dcaa65aab43ac917459d1054ec46d87ec81#diff-0bdb1ee2e8f9baa5430ac95a17eb3ad0a67c356a510ee3a50a236017d06bfc4a)**

## High Level Overview 
**TODO**
- Utilizes ADC's DMA Sequencing to auto-update inputs (I think).
- Utilizes the ADC Host-Client mode so ADC1 follows ADC0's configuration.
- Utilizes DMAC to gather results into 2 buffers - one per ADC.
	- **Channel 0**
		- 
	- **Channel 1**
		- 
	- **Channel 2**
		- Used for DMA Sequencing 
	- **Channel 3**
		- Used for grabbing ADC0's results
		- Circular; This descriptor points to Channel 0's descriptor
	- **Channel 4**
		- Used for grabbing ADC1's results
		- Circular; This descriptor points to Channel 1's descriptor


## Globals and Fluff
Code starts with SD setup. Each [[ADC]] gets its own buffer to store results. The buffer is then split between each pin. 

Then, a 4 element array is initialized with the pins for the ADC multiplexor. These are the input pins to be read from.

Next, a simple [[DMA Controller#^4baaaf|descriptor struct]] (`dmacdescriptor`) is declared.

Array pointers are made for the [[DMA Controller#^b2d110|writeback section]] and descriptor section to reside in SRAM. There's also a declaration for `descriptor`, which will be used to make copying descriptors over to the descriptor section easier. There's also a 2 element array of `dmacdescriptor`s made in preparation to link the two active ones together.

## setup()
The SD/SPI stuff is initialized with the parameters from above. Since the SPI library has its own DMAC setup, the following lines are used to realign the descriptor section and writeback section addresses to those set by the SPI library:

```cpp
descriptor_section =  reinterpret_cast<dmacdescriptor(*)[DMAC_CH_NUM]>(DMAC->BASEADDR.reg); //point array pointer to BASEADDR defined by SD.begin

    wrb =  reinterpret_cast<dmacdescriptor(*)[DMAC_CH_NUM]>(DMAC->WRBADDR.reg);                 //point array pointer to WRBADDR defined by SD.begin
```

The [[DMA Controller|DMAC]] is then enabled. At the same time, all priority levels are enabled via:
```cpp
    DMAC->CTRL.reg = DMAC_CTRL_DMAENABLE | DMAC_CTRL_LVLEN(0xf);
```
Referencing the datasheet and the definition for `DMAC_CTRL_LVLEN`, the latter term in the bitwise `or` operator sets the priority-level-enable bits all to `1`.

### ADC DMA Sequencing Setup
DMAC Channel 2's priority is set to 3 (highest). It's trigger source is set to ADC0's `DSEQ` trigger. The conditions for this trigger are shown in *Section 45.6.3.4* of the SAMD51 datasheet. When this trigger request comes, it triggers a *burst transfer*.

Next, the empty `descriptor` is configured. The next descriptor address is set to itself, creating a *circular descriptor*., and `dstaddr` is .  <span style="background:#fff88f">The descriptor is configured according to datasheet *Section 45.6.3.4* for DMA Sequencing.</span> The descriptor is then set to be *valid*.

`descriptor` is modified  in order to make copying easier:
- **descaddr:** Set to itself. Circular descriptor
- **srcaddr:**  Set to the end of the `inputCtrl0` array. For some reason, the 4 bytes after `inputCtrl0` are considered relevant as the sequencing control data.
- **dstaddr:** Set to ADC0's **DSEQDATA** register
- **btcnt:** Beat count is set to *4 beats*.
- **btctrl:** 
	- Beat size is set to a word (32 bits)
	- The source address is set to be *incremented*
	- Step Selection settings are applied to the *source address*
	- Step Size is set to *1 beat*

### DMAC ADC0 Setup
DMAC Channel 3 is set to priority level 3. It's trigger source is set to ADC0's `RESRDY` interrupt, which triggers when a ADC0 result is ready. Channel 3 is configured to do *burst transfers*.

`descriptor` is modified again in order to make copying easier:
- **descaddr:** Next descriptor is set to *Descriptor 0*'s address
- **srcaddr:** Source is set to ADC0's **RESULT** register
- **dstaddr:** Destination is set to the halfway point of `adcResults0`
- **btcnt:** Number of beats is 1024 for now; Each pin gets 1024 samples 
- **btctrl:** 
	- Beat size is set to a *half-word* (16 bits) which easily fits our 12 bit results
	- Set to increment the destination address; will fill up the buffer properly and not just overwrite an element over and over.
	- The Descriptor is set as *valid*
	- This channel (3) is set to suspend once the block transfer is completed.
This descriptor is then copied to Channel 3 of the *descriptor section.*

`descriptor` is modified again in order to make copying easier:
- **descaddr:** Next descriptor is set to *Descriptor 3*'s address - look above
- **srcaddr:** Source is set to ADC0's **RESULT** register
- **dstaddr:** Destination is set to the start of `adcResults0`
- **btcnt:** Number of beats is set to 1024.
- **btctrl:** 
	- Beat size is set to a *half-word* (16 bits) which easily fits our 12 bit results
	- Set to increment the destination address; will fill up the buffer properly and not just overwrite an element over and over.
	- The Descriptor is set as *valid*
	- This channel (0) is set to suspend once the block transfer is completed.
This descriptor is then copied to Channel 0 of the *descriptor section*.


### DMAC ADC1 Setup
DMAC Channel 4 is set to priority level 3. It's trigger source is set to ADC0's `RESRDY` interrupt, which triggers when a ADC1 result is ready. Channel 4 is configured to do *burst transfers*.

`descriptor` is modified again in order to make copying easier:
- **descaddr:** Next descriptor is set to *Descriptor 1's address
- **srcaddr:** Source is set to ADC1's **RESULT** register
- **dstaddr:** Destination is set to the halfway point of `adcResults1
- **btcnt:** Number of beats is 1024 for now; Each pin gets 1024 samples 
- **btctrl:** 
	- Beat size is set to a *half-word* (16 bits) which easily fits our 12 bit results
	- Set to increment the destination address; will fill up the buffer properly and not just overwrite an element over and over.
	- The Descriptor is set as *valid*
	- This channel (4) is set to suspend once the block transfer is completed.
This descriptor is then copied to Channel 4 of the *descriptor section.*

`descriptor` is modified again in order to make copying easier:
- **descaddr:** Next descriptor is set to *Descriptor 4*'s address - look above
- **srcaddr:** Source is set to ADC1's **RESULT** register
- **dstaddr:** Destination is set to the start of `adcResults1`
- **btcnt:** Number of beats is set to 1024.
- **btctrl:** 
	- Beat size is set to a *half-word* (16 bits) which easily fits our 12 bit results
	- Set to increment the destination address; will fill up the buffer properly and not just overwrite an element over and over.
	- The Descriptor is set as *valid*
	- This channel (1) is set to suspend once the block transfer is completed.
This descriptor is then copied to Channel 1 of the *descriptor section*.