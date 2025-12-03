---
author: Andrew Y
tags: Stub
---
  
![[Pasted image 20250107131911.png]]
https://docs.google.com/spreadsheets/d/1VJTuBukCeR8GGBbfJELEy8z72eeXJkKCebAghGk4s7Q/edit?usp=sharing
## Components
### Diode
- Charged particles *fly through* depleted region
	- leaves ionized trail
- Photons *absorbed* in depleted region
	- produces an electron
- **reverse bias**
	- *virtually no current!*
	- Great for **gamma ray detection**
		- charged particles create **brief current pulse** like a marker
- input is "dark current"
	- from thermal excitation on the diode
	- constant input
	- problem occurs if this baseline is too large bc you lose dynamic range on the op amps 0-3.3V range
### Op-Amp
- scintillation preamplifier?
- not a standard config (noninverting, differentiator, integrator)
- 0-3.3V range
- *Gain will amplify our baseline and any noise!*
### Capacitor
- **V = Q/C**
- capacitance controls sensitivity
- smaller capacitance = less sensitive
	- low C
	- high V
	- Op-Amp creates 
- larger capacitance = more sensitive (includes more of the baseline signal)
### Resistor
- **V = I\*R**
	- **I** is the leakage current from diode
- Resistance controls baseline, time constant
	- resistance *proportional* to baseline
- If **R** too high, baseline is too high, and our *usable range is smaller*
	- **less sensitive sensor!**
#### Time Constant
- Higher time constant
	- Slower decay
		- *Can sample slower!*
			- less burdensome on digital side
- Lower time constant
	- Faster decay
		- *Must sample faster!*