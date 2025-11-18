---
author: Andrew Y
---
## About
This is a guide on how to set up and operate the experimental IRIS GUI built on the Python Textual TUI framework. It can be operated with the mouse or with just the keyboard.

## Dependencies
- [uv](https://docs.astral.sh/uv/getting-started/installation/)
	- Using `uv python install`, install a recent version of Python (3.12+)
	- **Please double check uv's recognized Python installations by running `uv python list` in terminal!**
- [A **Local Copy** of the GUI](https://github.com/IRIS-Digital-Dosimeter/IRIS-GUI/tree/textual-attempt)
	- **Make sure to pull the `textual-attempt` branch**

### Running the GUI
1. Open a terminal window in the **IRIS-GUI repo** directory.
2. Run `uv run textual run textualtest.py`
	- User your mouse or keyboard to navigate the menus.
3. For MDA, preset sketches are included in the **MDAScreen**
#### Screens
The GUI is split into several "screens". You can switch between them with **CTRL+S**. You're also prompted at startup to choose a screen.
- For MDA, preset sketches are included in the **MDAScreen**
- For manual operation, the **Manual Upload Screen** is available
	- The included config editor is read-only. Your changes will not be reflected in the final product
