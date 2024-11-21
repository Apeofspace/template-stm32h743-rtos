Template for `NUCLEO-stm32h743zi2` to use with CMake and CubeMX.
In cube add all that you need. Select to only copy libs that you need (or all of them, whatev).
Select **basic** app structure, and toolchain `Makefile`.
Debug with VSCode, cuz I couldn't figure out how to setup DAP with neovim.
Double check `CMakeLists.txt` to perhaps remove some unused libs 

Use with something like
```bash
make -j10
make flash
```
