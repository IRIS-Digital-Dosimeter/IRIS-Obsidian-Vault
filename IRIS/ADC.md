

> [!tldr]- Datasheet Reference
> [[Resources#^d11da6|Section 45]]

The [[M4 Board|SAMD51]] has two 12-bit ADCs onboard. They each have a set of double-buffered registers used to control the ADCs' configurations. A lot of the requirements for usage are already defined by Adafruit's `wiring.c`, though it's not too clear which ones.


> [!info]- ADC Specs
> - Two ADCs on board
> - 8, 10, 12 bit resolution
> - Up to 1 million samples per second
> - Single-ended and Differential results
> - Internal inputs
> 	- Internal temp sensor
> 	- Scaled IO supply
> 	- Scalled VBAT supply
> 	- DAC0
> - Single, continuous, or sequenced conversion options
> - Window monitor with selectable channel
> - Event-triggered conversion for accurate timing
> - Result DMA transfer available
> - Gain and Offset compensation
> - Averaging and Oversampling+Decimation up to 16 bit resolution
> - Configurable sampling time (in clock cycles)
> - Client/Host configuration available
> 	- ADC1 can "copy" ADC0's configuration


> [!info]- Block Diagram
> ![[Pasted image 20240825191722.png]]



# General Notes
Conversion results are ready when **INTFLAG.RESRDY** is `1`.

In *free running mode*, the sampling rate R$_{S}$ is calculated by:
$$
R_S = \frac{f_{\text{CLK\_ADC}}}{n_{\text{SAMPLING}} + n_{\text{OFFCOMP}} + n_{\text{DATA}}} * \frac{1}{n_\text{SAMPLES}}
$$
where:
- n<sub>sampling</sub> is the total sampling duration in CLK_ADC cycles
- n<sub>offcomp</sub> is the offset compensation in CLK_ADC cycles
- n<sub>data</sub> is the resolution of the ADC (8, 10, 12)
- n<sub></sub>
- f<sub>CLK_ADC</sub> = $f_{\text{GCLK\_ADC}} / 2^{(1 + \text{CTRLA.PRESCALER})}$

# Configuration

> [!note] Sample Distribution
> The goal is to spread out the sampling such that X samples are evenly distributed within the time it takes to write X samples to the SD card. 
> 
> Essentially, we're looking for spaced out datapoints across 1000us rather than a burst of data every 1000us.

Currently, I'm looking at using the following configurations:
## Prescaler Selection
For adjusting sampling rate. Low priority. Adjusted via **CTRLA** register.

## Single-Ended Mode Over Differential mode
Since we will always have positive values, we will use *single ended mode*. **INPUTCTRL.DIFFMODE** must be set to `1`.

## Sampling Time
How many ADC clock cycles (CLK_ADC) are used per conversion. When **SAMPCTRL.SAMPLEN** is set to `0`, then only one clock cycle is used. Can be used to adjust sampling rate/accuracy(?).

## Averaging Mode
For adjusting sampling rate and accuracy. It accumulates 2<sup>n</sup> samples and then bit-shifts down to 12 bits for the averaging property. **AVGCTRL** should be configured according to the following table:
![[Pasted image 20240824135815.png]]

## DMA Sequencing
The ADC can sequence multiple conversions. When using sequencing, the ADC's configuration registers can be updated by the [[DMA Controller]].

> [!NOTE] 
> In any of the accumulation/averaging modes, the next sequenced conversion starts when the averaged result is ready.


> [!info]- Diagram
> ![[Pasted image 20240825192645.png]]


> [!warning] 
> Free running mode cannot be used with DMA Sequencing.
> If a conversion is triggered by event (**EVCTRL.STARTEI** is `1`) then automatic start conversion is disabled.


Enabled by setting any field in **DSEQCTRL** to `1`. When turned on, **DSEQSTAT.BUSY** is set to `1`. By enabling a register via **DSEQCTRL**, the corresponding ADC register is updated automatically. 

By default, a new conversion will start when a trigger is received. However, <span style="background:rgba(3, 135, 102, 0.2)">writing a `1` to **DSEQCTRL.AUTOSTART** will automatically start a new conversion when a DMA sequence completes.</span>

The [[DMA Controller|DMAC]] must be configured with the following:
![[Pasted image 20240825152715.png]]![[Pasted image 20240825152745.png]]

## Host-Client/Master-Slave Operation
`ADC0` can be set as a *host* to `ADC1` such that `ADC1` uses the same configuration as `ADC0`. This is done by by writing a `1` to **ADC1.CTRLA.SLAVEEN**. When enabled, `GCLK_ADC0` clock and `ADC0` controls are internally routed to `ADC1`.


> [!info]- Diagram
> ![[Pasted image 20240825192804.png]]
