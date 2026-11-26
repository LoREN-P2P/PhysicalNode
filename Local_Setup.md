# Seeed Wio-E5 — Local Development

This guide describes how to compile and flash the `Seeed-LoRa-E5_PingPong` firmware locally on macOS using the ARM GNU toolchain, Make, STM32CubeProgrammer and an external ST-Link programmer. STM32CubeIDE is used for initial project configuration and build file generation, but does not need to be open during normal development.

## 1. Prerequisites

- macOS (Apple Silicon for the toolchain path shown below)
- STM32CubeIDE
- STM32CubeProgrammer
- Seeed Wio-E5 Mini (STM32WLE5JCI6)
- ST-Link programmer
- GNU Make and macOS command-line tools
- Python 3 and `compiledb` (optional, for clangd)

Install macOS command-line tools if needed:

```sh
xcode-select --install
```

## 2. Locate the compiler and programmer, and configure PATH

STM32CubeIDE bundles both the ARM GNU compiler and STM32CubeProgrammer CLI. Find the executables before compiling or flashing:

```sh
find /Applications/STM32CubeIDE.app -name arm-none-eabi-gcc 2>/dev/null
find /Applications/STM32CubeIDE.app -name STM32_Programmer_CLI 2>/dev/null
```

On the current Apple Silicon installation, these commands return:

```text
/Applications/STM32CubeIDE.app/Contents/Eclipse/plugins/com.st.stm32cube.ide.mcu.externaltools.gnu-tools-for-stm32.14.3.rel1.macosaarch64_1.0.0.202602081740/tools/bin/arm-none-eabi-gcc
/Applications/STM32CubeIDE.app/Contents/Eclipse/plugins/com.st.stm32cube.ide.mcu.externaltools.cubeprogrammer.macosaarch64_1.0.100.202603051304/tools/bin/STM32_Programmer_CLI
```

### Add both tools to PATH

For Zsh or Bash:

```sh
export PATH="/Applications/STM32CubeIDE.app/Contents/Eclipse/plugins/com.st.stm32cube.ide.mcu.externaltools.gnu-tools-for-stm32.14.3.rel1.macosaarch64_1.0.0.202602081740/tools/bin:$PATH"
export PATH="/Applications/STM32CubeIDE.app/Contents/Eclipse/plugins/com.st.stm32cube.ide.mcu.externaltools.cubeprogrammer.macosaarch64_1.0.100.202603051304/tools/bin:$PATH"
```

To make these changes permanent, place both export lines in `~/.zshrc` (Zsh) or `~/.bashrc` (Bash), then reload the corresponding file (e.g. `source ~/.zshrc`).

For Fish, use the following commands instead. By default, `fish_add_path` persists additions across sessions:

```fish
fish_add_path /Applications/STM32CubeIDE.app/Contents/Eclipse/plugins/com.st.stm32cube.ide.mcu.externaltools.gnu-tools-for-stm32.14.3.rel1.macosaarch64_1.0.0.202602081740/tools/bin
fish_add_path /Applications/STM32CubeIDE.app/Contents/Eclipse/plugins/com.st.stm32cube.ide.mcu.externaltools.cubeprogrammer.macosaarch64_1.0.100.202603051304/tools/bin
```

Always add the **containing directory (`tools/bin`)**, not the executable itself, to `PATH`. The versioned directory names may change after a CubeIDE update; rerun the `find` commands and adjust the paths accordingly.

### Verify both tools

```sh
command -v arm-none-eabi-gcc
command -v STM32_Programmer_CLI
arm-none-eabi-gcc --version
STM32_Programmer_CLI --help
make --version
```

## 3. Compile the firmware locally

Open a terminal at the firmware project root. STM32CubeIDE must have generated the `Debug/` build directory and its Makefiles first. If they are missing, import the project into CubeIDE and build the Debug configuration once.

```sh
cd Debug
make -j8 all
```

`-j8` allows up to eight parallel build jobs. To clean and rebuild:

```sh
make clean
make -j8 all
```

The generated build outputs include:

| File | Description |
| --- | --- |
| `Seeed-LoRa-E5_PingPong.elf` | Executable firmware with debug information |
| `Seeed-LoRa-E5_PingPong.hex` | Intel HEX firmware image |
| `Seeed-LoRa-E5_PingPong.map` | Linker memory map |
| `Seeed-LoRa-E5_PingPong.list` | Disassembly/listing |

The ELF file is used for flashing in this guide.

## 4. Connect the ST-Link

The USB-C port on the Wio-E5 Mini provides USB-to-UART communication; it is not an onboard SWD programmer. Use an external ST-Link to program native STM32 firmware.

| ST-Link signal | Wio-E5 signal |
| --- | --- |
| SWDIO | PA13 / SWDIO |
| SWCLK | PA14 / SWCLK |
| GND | GND |
| VTref (target voltage reference, if supported) | 3.3 V target reference |
| NRST (optional, useful for recovery) | NRST |

Power the target appropriately and ensure a common ground. Do not connect a 5 V supply to a 3.3 V target or assume that the ST-Link voltage reference pin supplies power.

## 5. Test the programmer connection

After configuring both tools in Section 2, connect the ST-Link and test the SWD connection:

```sh
STM32_Programmer_CLI -c port=SWD
```

## 6. Flash the firmware

From the `Debug/` directory, run:

```sh
STM32_Programmer_CLI -c port=SWD -w Seeed-LoRa-E5_PingPong.elf -v -rst
```

| Argument | Meaning |
| --- | --- |
| `-c port=SWD` | Connect via Serial Wire Debug |
| `-w` | Write/program the firmware |
| `-v` | Verify programmed contents |
| `-rst` | Reset the MCU |

An ELF file contains its load addresses, so no explicit flash address is needed. For a raw binary, the usual internal flash base address for this MCU is `0x08000000`.

From the project root, compile and flash in one command:

```sh
make -C Debug -j8 all && STM32_Programmer_CLI -c port=SWD -w Debug/Seeed-LoRa-E5_PingPong.elf -v -rst
```

The `&&` operator prevents flashing if compilation fails.

## 7. Configure clangd (optional)

To obtain code completion, diagnostics and navigation outside CubeIDE, generate a compilation database from the Make build.

Install `compiledb` (for example, using pipx):

```sh
pipx install compiledb
```

From the project root:

```sh
cd Debug
make clean
compiledb make -j8 all
cd ..
cp Debug/compile_commands.json .
```

The compilation database records the actual compiler arguments, include directories, preprocessor definitions and working directories.

Create a `.clangd` file in the project root:

```yaml
CompileFlags:
  CompilationDatabase: .
  Remove:
    - -fcyclomatic-complexity
```

If clangd cannot find standard headers such as `string.h`, configure the editor's clangd launch arguments to allow querying the actual ARM compiler:

```text
--query-driver=/Applications/STM32CubeIDE.app/Contents/Eclipse/plugins/com.st.stm32cube.ide.mcu.externaltools.gnu-tools-for-stm32.14.3.rel1.macosaarch64_1.0.0.202602081740/tools/bin/arm-none-eabi-gcc
```

For example, with the VS Code clangd extension, add this argument to `clangd.arguments` in the editor settings. Restart clangd after changing the configuration. Regenerate the compilation database whenever include paths, source files or compiler flags change.

## 8. Verify the firmware

Connect the Wio-E5 Mini via USB-C and locate its serial device:

```sh
ls /dev/cu.*
```

Open the appropriate serial device using the baud rate configured in the firmware (for example, 115200 if configured):

```sh
screen /dev/cu.usbserial-XXXX 115200
```

Replace the device name with the actual device. Exit `screen` using `Ctrl-A`, then `K` and confirm.

The PingPong firmware should print startup information similar to:

```text
PING PONG APPLICATIONVERSION: V1.3.0
MWRADIOVERSION: V1.3.0
LORABW=125 kHz
LORASF=7
```

For the two-node test, ensure that both devices have matching radio settings, including the configured 915 MHz frequency, and appropriate antennas. Observe the serial output on each device to verify packet exchange.

## 9. Troubleshooting

| Problem | Solution |
| --- | --- |
| `arm-none-eabi-gcc: command not found` | Add the compiler's containing directory to `PATH`. |
| `STM32_Programmer_CLI: command not found` | Add the CubeProgrammer CLI directory to `PATH`. |
| No Makefile found | Run Make from `Debug/`, or generate the build files in CubeIDE. |
| clangd reports `Unknown argument: -fcyclomatic-complexity` | Remove the flag in `.clangd`, not from the GCC build. |
| clangd cannot find `string.h` | Configure `--query-driver` with the ARM GCC executable and check the compilation database. |
| ST-Link cannot connect | Check power, common ground, SWD wiring and target voltage; try a lower SWD frequency. |

Example of flashing with a lower SWD clock frequency (950 kHz):

```sh
STM32_Programmer_CLI -c port=SWD freq=950 -w Debug/Seeed-LoRa-E5_PingPong.elf -v -rst
```

Some factory Wio-E5 firmware may have readout protection enabled. Changing RDP from level 1 (`0xBB`) to level 0 (`0xAA`) triggers a mass erase, destroying the existing firmware. Only perform this operation if deliberately replacing factory firmware and after checking the target's option bytes in STM32CubeProgrammer. Do not change RDP to level 2.

## 10. Daily development workflow

Once the initial setup is complete, from the project root:

```sh
# Compile
make -C Debug -j8 all

# Flash, verify and reset
STM32_Programmer_CLI -c port=SWD -w Debug/Seeed-LoRa-E5_PingPong.elf -v -rst

# Or compile and flash together
make -C Debug -j8 all && STM32_Programmer_CLI -c port=SWD -w Debug/Seeed-LoRa-E5_PingPong.elf -v -rst
```

STM32CubeIDE is only needed again when changing project configuration or regenerating its managed build files.
