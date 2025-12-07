---
author: Andrew Y
---
## Requirements
- M4 **or** M0 board
## Overview
This sketch constantly dumps samples from pins **A0** and **A1** over the serial line. Data can be captured/processed via the `serial_importer_all_in_one` script.

The included Python script `serial_importer_all_in_one.py` can be used to collect, print, and plot the incoming data. 