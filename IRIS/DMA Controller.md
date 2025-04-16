---
aliases:
  - DMAC
  - DMA
tags:
  - Stub
---
*Many paragraphs were paraphrased or lifted straight from the datasheet!*

# General Information


> [!tldr]- Datasheet Reference
> [[Resources#^d11da6|Section 22]]

The SAMD51's  *DMAC* contains both DMA capabilities and a CRC engine (which we don't use). The *DMAC* can move data across RAM and peripherals without [[M4 Board|CPU]] intervention. This lets us simultaneously collect [[20 - Professional/21 - SCIPP/21.01 - IRIS/IRIS-Notes-Site/docs/ADC|ADC]] samples while writing those samples to SD, allowing a constant collection rate.

The DMA engine works with several *DMA channels*, with only one being considered the *active channel* at any time. These channels can receive *transfer triggers* which generate *transfer requests*. These requests are handled by the *arbiter*, which decides which channel will be *active*. This will then fetch a user-defined *descriptor* for the active channel, and then finally execute the data transfer. A *descriptor* describes how a block transfer should be carried out by the DMAC. 

These *descriptors* must be stored in a contiguous block of SRAM, starting from the address written to the **BASEADDR** (*descriptor memory section*) and **WRBADDR** (*write-back memory section*) registers. The *descriptor memory section* must hold the first transfer descriptors ordered by channel number. **BASEADDR** therefore points only to the first transfer descriptor of channel 0.

**All of this is described in further detail below!**

> [!info]+
> Note that the DMAC's internal memory only holds the currently active descriptor's details. This is why there's a writeback - to keep the current descriptor in memory somewhere when the DMAC switches to another descriptor.
> 

^b2d110

> [!tip]+
> Note that the DMAC's other registers (e.g. all the **CHTRLAx** and **CHTRLBx** registers) are per-channel.

> [!NOTE]- DMAC Specs
> - Trigger Sources:
> 	- Software
> 	- Events
> 	- Dedicated Peripheral Requests
> - Linkable/Circular descriptors
> - 32 Available Channels
> 	- Priority levels (1-4 ascending) with fixed or RR scheduling
> 	- Preemptive 
> - Transfer 1KB to 256KB in one block transfer
> - Can generate interrupts on:
>	- block transfer completion
>	- error detection
>	- channel suspension
>- CRC polynomial selectable to
>	- CRC-16
>	- CRC-32

> [!info]- Block Diagram 
> ![[Pasted image 20240825191625.png]]

> [!info]+ Common Vernacular
> - TODO!
> -  **Transfer Descriptor**
> 	- Describes how a block transfer should be executed
> - **DMA Transaction**
> 	- A complete IO operation (read/write request)
> 	- e.g. An array was moved from A to B
> - **DMA Transfer**
> 	- A single operation by hardware that transfers some data
> 	- *DMA Transactions* consist of one or more *DMA Transfers*
> - **Beat Transfer**
> 	- The size of one unit of data transfer, where the size of the *beat* is defined by writing to **BTCTRL.BEATSIZE**
> - **Block Transfer**
> 	- The amount of data a single *transfer descriptor* can transfer. Ranges from 1 to 64,000 beats. Interruptable.
> - **Burst Transfer**
> 	- Back-to-back beat transfers without CPU interference.
> 	- <span style="background:#fff88f">Educated Guess</span>: block transfers are comprised of burst transfers
> - **Transaction**
> 	- The *DMAC* can link several transfer descriptors by having the first descriptor point to the second, and so forth. A *DMA Transaction* is the complete transfer of all blocks within a linked list.
> 	- ![[Pasted image 20240831140810.png]]

# DMAC Initialization
## General Initialization
Before enabling the DMAC, the **BASEADDR** and **WRBADDR** registers must be populated. These registers are *enable protected*, meaning they can only be set when the DMAC is disabled. 

## DMA Channel Initialization
Before enabling a *channel*, the channel and the corresponding *first transfer descriptor* must be configured.

### Channel Configuration
- To configure channel *x* (0-31), the corresponding register **CHCTRLA*x*** must be populated.
- One of the following *trigger actions* must be written to **CHCTRLAx.TRIGACT**:
	 ![[Pasted image 20240831135025.png]]
- A *trigger source* must be written to **CHCTRLAx.TRGSRC**. The sources can be found in section **22.8.16** of the [[Resources#^d11da6|SAMD51 Datasheet]].

### Transfer Descriptor Configuration

^4baaaf

> [!tip] A descriptor can be stored as a simple struct:
> ```cpp
> typedef struct
> {
> 	uint16_t btctrl;
> 	uint16_t btcnt;
> 	uint32_t srcaddr;
> 	uint32_t dstaddr;
> 	uint32_t descaddr;
> } descriptor;
> ```

- The *Beat Size* must be configured in the descriptor's **BTCRL.BEATSIZE**: 
	![[Pasted image 20240831145149.png]]
- The *descriptor* must be be made valid by writing a `1` to the descriptor's **BTCTRL.VALID** field.
- The *number of beats per block transfer* must be set by writing to the descriptor's **BTCNT** field.
- The *source address* for the block transfer must be set by writing to the descriptor's **SRCADDR** field.
- The *destination address* for the block transfer must be set by writing to the descriptor's **DSTADDR** field.
- The *address of the next descriptor* must be set in the descriptor's **DESCADDR** field. *If it is the last descriptor in the linked list, then this must be set to **0x0***

## Enabling the DMAC and Channels
The DMAC will only function properly if a channel is configured properly and if a descriptor is configured properly before enabling. The DMAC is enabled by writing a `1` to **CTRL.DMAENABLE**, and disabled by writing a `0`. Once configured, a DMA channel can be enabled by writing a `1` to the corresponding **CHCTRLAx.ENABLE** bit. 

# Details of Operation


> [!warning]
> The DMAC can only perform a data transmission if the following criteria are met:
> 1. A DMA channel *x* has to be configured and enable.
> 2. Channel *x* must have an initialized transfer descriptor.
> 3. The *arbiter* has to appoint channel *x* as the *active channel*.

## General Operation
If a DMA channel is enabled and not suspended, and it receives a *transfer trigger*, it will send a *transfer request* to the *arbiter*. The arbiter will then include the channel in a queue of channels with pending transfers, which will be reflected in the **PENDCH.PENDCHx** register. Based on the arbitration scheme (static or dynamic), the arbiter will pick the *next active channel* (it's authorized to perform its next *burst transfer* once the current burst completes). 

The corresponding descriptor will be fetched and stored in the *Pre-Fetch Channel*. Once the current active channel completes its burst, the *next active channel*'s descriptor is moved from the *Pre-Fetch Channel* to the *Active Channel*, and the burst transfer begins. When this move happens, the corresponding **PENDCH.PENDCHx** bit will be cleared. The **BUSYCH.BUSYCHx** register will also be updated accordingly.

Once the channel has performed its granted burst transfer it will either be put back into the queue of channels with pending transfers, set to be waiting for a new transfer trigger, suspended, or disabled. This depends on the channel and block transfer configuration. **BUSYCH.BUSYCHx** will be updated accordingly.


> [!NOTE]
> - If a DMA channel is suspended while it has a pending transfer, it will be removed from the queue of pending channels.
> - If a DMA channel gets disabled while it has a pending transfer, it will be removed from the queue of pending channels.
> - If a DMA channel still has pending transfers, it will be returned to the pending queue.

## Priority Levels
Each DMA channel supports up to 4 priority levels. The *priority level* for a channel *x* is set by writing to **CHPRILVLx.PRILVL**..When all priority levels are enabled, channels with higher priority values will take priority over those with lower priority values. 


> [!NOTE] Enabling Priority Levels & Scheme
> Priority level *x* is enabled by enabling the corresponding bit in **CTRL.LVLENx**.
> 
> The DMAC can be configured to use *static*(0) or *dynamic*(1) arbitration schemes for priority *x* by writing the corresponding bit to **PRICTRL0.RRLVLENx**.
> - Static Arbitration means the arbiter will prioritize a lower channel number over a higher channel number of the same priority level. 
> 	- In this scheme, high channel numbers have a chance to never be granted access as *the active channel*.
> -  Dynamic Arbitration means the arbiter will use *Round Robin Scheduling* for that priority level.

