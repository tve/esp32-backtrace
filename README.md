ESP32 Exception Stack Backtrace Analyzer
========================================

This little bash script parses the exception backtrace printed by the ESP32 firmware
and uses GDB to print out the source location of all code addresses.

It is heavily inspired by https://github.com/me-no-dev/EspExceptionDecoder but only uses
bash and xtensa-esp32-elf GDB.

__Install:__ Place the esp32-backtrace script somewhere in your path or use the full pathname to run
it. If you are not using platformio to build your project you need to edit the `XTENSA_GDB`
definition at the top of the script.

Example
-------
My program crashed printing:
```
Guru Meditation Error: Core  0 panic'ed (Unhandled debug exception)
Debug exception reason: Stack canary watchpoint triggered (Tmr Svc)
Core 0 register dump:
PC      : 0x40081708  PS      : 0x00060336  A0      : 0x3ffbc990  A1      : 0x3ffbc8d0
A2      : 0x3ffb7dd0  A3      : 0x00000000  A4      : 0x3ffb7e18  A5      : 0x00000000
A6      : 0x00000001  A7      : 0x00000018  A8      : 0x800894ac  A9      : 0x3ffbc970
A10     : 0x00000000  A11     : 0x3ffc12c8  A12     : 0x3ffc12c8  A13     : 0x00000001
A14     : 0x00060320  A15     : 0x00000000  SAR     : 0x00000002  EXCCAUSE: 0x00000001
EXCVADDR: 0x00000000  LBEG    : 0x40001609  LEND    : 0x4000160d  LCOUNT  : 0x00000000

Backtrace: 0x40081708:0x3ffbc8d0 0x3ffbc98d:0x3ffbc9d0 0x400d85b6:0x3ffbc9f0 0x40082926:0x3ffbca10 0
x400822fe:0x3ffbca30 0x400dd591:0x3ffbcaa0 0x400dc019:0x3ffbcac0 0x400dc467:0x3ffbcae0 0x400dc5f5:0x
3ffbcb50 0x400db105:0x3ffbcbd0 0x400db1b4:0x3ffbcc50 0x400da27e:0x3ffbccb0 0x400da7b1:0x3ffbccf0 0x4
01396b2:0x3ffbcd10 0x401398a6:0x3ffbcd60 0x401398f9:0x3ffbcd90 0x401017ec:0x3ffbcdb0 0x401019fa:0x3f
fbcde0 0x400d9721:0x3ffbce10 0x400
```
I use platformio so the elf file for my application is in
`.pio/build/{environment}/{project}.elf`
so I launched the esp-backtrace as:
`esp32-backtrace .pio/build/usb/esp32-secure-base.elf`
and I pasted the crash printout into its stdin and the result is:
```
PC: 0x40081708 is at esp-idf-public/components/freertos/xtensa_vectors.S:1118.
BT-0: 0x40081708 is at esp-idf-public/components/freertos/xtensa_vectors.S:1118.
BT-2: 0x400d85b6 is in esp_ipc_call (esp-idf-public/components/esp32/ipc.c:123).
BT-3: 0x40082926 is in spi_flash_disable_interrupts_caches_and_other_cpu (esp-idf-public/components/
spi_flash/cache_utils.c:120).
BT-4: 0x400822fe is in spi_flash_read (esp-idf-public/components/spi_flash/flash_ops.c:154).
BT-5: 0x400dd591 is in nvs::nvs_flash_read(unsigned int, void*, unsigned int) (esp-idf-public/compon
ents/nvs_flash/src/nvs_ops.cpp:70).
BT-6: 0x400dc019 is in nvs::Page::readEntry(unsigned int, nvs::Item&) const (esp-idf-public/componen
ts/nvs_flash/src/nvs_page.cpp:717).
BT-7: 0x400dc467 is in nvs::Page::findItem(unsigned char, nvs::ItemType, char const*, unsigned int&,
 nvs::Item&, unsigned char, nvs::VerOffset) (esp-idf-public/components/nvs_flash/src/nvs_page.cpp:76
1).
BT-8: 0x400dc5f5 is in nvs::Page::readItem(unsigned char, nvs::ItemType, char const*, void*, unsigne
d int, unsigned char, nvs::VerOffset) (esp-idf-public/components/nvs_flash/src/nvs_page.cpp:266).
BT-9: 0x400db105 is in nvs::Storage::readMultiPageBlob(unsigned char, char const*, void*, unsigned i
nt) (esp-idf-public/components/nvs_flash/src/nvs_storage.cpp:430).
BT-10: 0x400db1b4 is in nvs::Storage::readItem(unsigned char, nvs::ItemType, char const*, void*, uns
igned int) (esp-idf-public/components/nvs_flash/src/nvs_storage.cpp:455).
BT-11: 0x400da27e is in nvs_get_str_or_blob(nvs_handle, nvs::ItemType, char const*, void*, size_t*)
(esp-idf-public/components/nvs_flash/src/nvs_api.cpp:515).
BT-12: 0x400da7b1 is in nvs_get_blob(nvs_handle, char const*, void*, size_t*) (esp-idf-public/compon
ents/nvs_flash/src/nvs_api.cpp:525).
BT-18: 0x400d9721 is in esp_wifi_init (esp-idf-public/components/esp32/wifi_init.c:51).
BT-24: 0x4008b3ba is in prvProcessExpiredTimer (esp-idf-public/components/freertos/timers.c:524).
BT-25: 0x4008b3ed is in prvProcessTimerOrBlockTask (esp-idf-public/components/freertos/timers.c:571)
.
BT-26: 0x4008b508 is in prvTimerTask (esp-idf-public/components/freertos/timers.c:544).
BT-27: 0x40088781 is in vPortTaskWrapper (esp-idf-public/components/freertos/port.c:143).
```
This tells me that the problem happens in an Arduino system task somewhere in
`esp_wifi_init` (see BT-18).

### Notes:
- the backtrace shows the inner-most frame first (BT-0) and then goes back in history.
- instead of pasting the exception text on stdin you can also specify a file to read from as second
  argument.
- it is OK to have the "Backtrace:" line from the exception text be split up due to copy&paste
  line-wrapping caused by the terminal emulator, the script joins them back together.

Issues
------
- While the script handles `Backtrace` continuation lines it doesn't join addresses that get split
  at the end of the terminal window.
- GDB is invoked for each address, which is inefficient. It works ok for me, but it would be more
  elegant to construct a script for GDB and pipe that in so GDB only reads and parses the elf file
  once.
- Anything that looks like an address in a register will be printed, whether it's live data or some
  stale value.
