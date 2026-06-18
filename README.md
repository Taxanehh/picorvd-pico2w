# PicoRVD (Pico 2 W / RP2350 port)

This is a fork of [aappleby/picorvd](https://github.com/aappleby/picorvd) that runs on the Raspberry Pi Pico 2 W (RP2350). All the real work is Austin Appleby's. This fork just makes the SWIO timing portable so it works on the newer board.

PicoRVD is a GDB-compatible remote debugger for the RISC-V CH32V003 chips from WinChipHead. It lets you program and debug a CH32V003 over WCH's single-wire SWIO interface without the proprietary WCH-Link dongle or a patched copy of OpenOCD. It runs on a Pi Pico and talks to the target chip over a single GPIO.

**Still very alpha. Expect bugs. Read the "Known issue" section below before you rely on it.**

## Why this fork exists

Stock PicoRVD hardcodes the PIO clock divider assuming the RP2040's 125 MHz system clock. The Pico 2 W (RP2350) boots at 150 MHz, so SWIO ended up running about 17 percent too fast. The stop bit dropped below the target's sampling threshold, the chip never ACKed, and you would see either all 0xFFFFFFFF reads or GDB just hanging on connect.

The fix is small: derive the divider from the live system clock instead of assuming a fixed one. Now it works on both the RP2040 and the RP2350.

What changed, all in `src/PicoSWIO.cpp`:

- The SWIO bit clock is computed from `clock_get_hz(clk_sys)` instead of a hardcoded constant, so the bit timing is correct at any system clock.
- The reset pulse uses `busy_wait_us()` instead of a hardcoded busy-loop count, so it does not drift when the core clock changes.

## Known issue (please read)

On the RP2350 the SWIO **write** path still has a PIO timing bug. Word writes execute, but the chip stores a fixed garbage value no matter what data you send. Reads, on the other hand, are rock solid.

So on a Pico 2 W:

- Reading RAM, CPU registers, and flash (when the chip is not read-protected) works great.
- Writing RAM or flash, and anything that depends on it (loading a binary with `load`, the flash routines, software breakpoints), is unreliable.

If you only need to read a target, this fork is ready to go. If you need writes, either fix the `op_write` timing in the PIO program or use an original RP2040 Pico for now. Patches very welcome.

## Hardware hookup

- CH32V003 PD1 (SWIO) to Pico GP28.
- CH32V003 GND to Pico GND.
- Pull-up resistor from SWIO to 3.3V.

About that pull-up: the original suggests 1k. On the Pico 2 W you will probably need something stiffer, around 5k or stronger. The RP2350 input pad has a known latch (erratum E9) that can hold the line near 2.1V, and a weak pull-up is not enough to pull it cleanly high, so the link never comes up. Two 10k resistors in parallel (so about 5k) worked here where a single 10k did not. If the link will not establish, lower the pull-up value before assuming anything else is wrong.

Power the CH32V003 however you like, either from its own supply or from the Pico's 3.3V rail.

## Usage

```
gdb-multiarch your_binary.elf
target extended-remote /dev/ttyACM0
```

Replace `ttyACM0` with whatever port your Pico shows up as. `monitor reset` resets the target. Keep the write caveat above in mind if you try `load`.

Reads are cached on the Pico side and redundant debug-register accesses are skipped, so most read-heavy operations are quicker than a stock WCH-Link.

## Building

Install the prerequisites:

```
sudo apt install cmake gcc-arm-none-eabi gcc-riscv64-unknown-elf xxd
```

Build it:

```
cmake -B bin
make -j -C bin
```

CMake auto-fetches the Pico SDK as part of the build. Two things matter for the Pico 2 W:

- You need a Pico SDK that supports the RP2350 (SDK 2.x or newer).
- You must build for the Pico 2 W board (`PICO_BOARD=pico2_w`), since the RP2350 is a different core than the RP2040. A binary built for the plain `pico` board will not run on it.

Flashing: hold BOOTSEL, plug the Pico in, and copy `bin/picorvd.uf2` to the drive that appears. Or run `upload.sh` if you have it connected through a Pico Debug Probe.

## Modules

PicoRVD is split into a few modules that can in principle be reused on their own. These are unchanged from upstream.

### PicoSWIO
Implements the WCH SWIO protocol using the Pico's PIO block. Exposes a simple get(addr) / put(addr, data) interface. No "fast mode" yet, but standard mode runs at around 800 kbps which is plenty. This is the file this fork patched for RP2350 timing.

Spec: https://github.com/openwch/ch32v003/blob/main/RISC-V%20QingKeV2%20Microprocessor%20Debug%20Manual.pdf

### RVDebug
Exposes the registers in the official RISC-V debug spec, plus methods to read and write memory over the main bus and to halt, resume, and reset the CPU.

Spec: https://github.com/riscv/riscv-debug-spec/blob/master/riscv-debug-stable.pdf

### WCHFlash
Methods to read and write the CH32V003's flash. A lot is hardcoded for now. WCHFlash does not clobber device RAM; it streams data straight to the flash page buffer, so in theory you can replace flash contents without resetting the CPU (untested upstream, and remember writes are broken on the RP2350 in this fork).

CH32V003 reference manual: http://www.wch-ic.com/downloads/CH32V003RM_PDF.html

### SoftBreak
The CH32V003 has no hardware breakpoints. The official WCH-Link fakes them by patching and unpatching flash on every halt and resume. SoftBreak does the same idea but minimizes page updates, and it skips them entirely during GDB's "single-step by putting a breakpoint on every instruction" pattern, which makes stepping much faster.

### GDBServer
Talks to the GDB host over the Pico's USB serial port and translates the GDB remote protocol into RVDebug, WCHFlash, and SoftBreak calls.

Spec, see Appendix E: https://sourceware.org/gdb/current/onlinedocs/gdb.pdf

### Console
A small serial console on UART0 (GP0/GP1) with commands for poking at the target and debugging the debugger itself. Connect with `minicom -b 1000000 -D /dev/ttyACM0` (swap in your port) and type `help`.

## Credit and license

Forked from [aappleby/picorvd](https://github.com/aappleby/picorvd) by Austin Appleby. Same license as upstream. If you find the RP2350 changes useful, consider sending them back upstream as a pull request.
