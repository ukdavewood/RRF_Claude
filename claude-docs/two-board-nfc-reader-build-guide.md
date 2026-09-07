# Building a 2-reader NFC filament sensor (ESP32 + 2x PN5180)

This guide covers wiring two PN5180 NFC reader modules to a single ESP32
dev board, and getting an AI coding assistant (Claude, ChatGPT, etc.) to
write the firmware for you. It assumes no prior experience with either.

## What you're building

An ESP32 that continuously scans two NFC readers for filament spool tags —
both Bambu Lab's Mifare-style tags and the open OpenPrintTag standard (ICODE
SLIX2) — and serves a small web dashboard showing what's currently on each
reader.

## Parts

- 1x ESP32 dev board (any standard 30/38-pin "ESP32 DevKit" style board)
- 2x PN5180 NFC reader modules (breakout boards with SPI + IRQ/BUSY/RST
  pins broken out — widely sold on AliExpress/eBay as "PN5180 NFC module")
- Breadboard/wire or a soldered header cable per reader (9 wires per
  reader: 3.3V, 5V, GND, NSS, BUSY, RST, SCK, MOSI, MISO)
- A USB power source able to comfortably run two RF front-ends at once —
  a powered USB hub is strongly recommended over a laptop USB port or an
  unpowered hub, since brief RF transmit current spikes on both readers can
  brown out a marginal supply

## Wiring

Both PN5180 modules **share** the SPI clock/data lines (SCK, MOSI, MISO) —
only one reader can be active on the bus at a time, which the firmware
handles. Each reader gets its **own** NSS (chip select), BUSY, and RST pin
so the ESP32 can address them independently.

| Signal | Board 1 | Board 2 | Shared? |
|---|---|---|---|
| NSS (chip select) | GPIO4  | GPIO32 | No — one per board |
| BUSY               | GPIO35 | GPIO21 | No — one per board |
| RST                | GPIO26 | GPIO33 | No — one per board |
| SCK                | GPIO25 | GPIO25 | Yes |
| MOSI                | GPIO23 | GPIO23 | Yes |
| MISO                | GPIO34 | GPIO34 | Yes |
| 3.3V | 3.3V | 3.3V | Yes |
| 5V (RF supply, if the module needs it) | 5V | 5V | Yes |
| GND | GND | GND | Yes |

Notes:
- GPIO34/35 are input-only pins on the ESP32, which is fine since MISO and
  BUSY are both ESP32-inputs — don't use them for NSS/RST (which are
  outputs).
- Avoid the ESP32's boot-strapping pins (GPIO0, 2, 5, 12, 15) for any
  signal the PN5180 or your own code drives during boot, to avoid
  interfering with boot mode selection. None of the pins above are strapping pins.
- Give each reader's RF antenna some physical separation if you can — two
  antennas mounted very close together can detune each other.

## Getting the firmware written

Rather than writing the firmware by hand, give the prompt below to Claude
or ChatGPT. It encodes the design constraints that matter for this specific
setup (shared-bus multi-reader scanning, dual tag-format support, and a
couple of known gotchas) so the assistant doesn't have to rediscover them.
Adjust it freely — remove the second tag format if you only need one, swap
in your own tag standard, or add more readers by extending the same
pattern.

> Write ESP-IDF (C, not Arduino) firmware for an ESP32 that reads NFC tags
> from two PN5180 modules sharing one SPI bus. Requirements:
>
> - Each PN5180 has its own NSS, BUSY, and RST GPIO; SCK/MOSI/MISO are
>   shared across both. Poll them in a round-robin loop, one full scan of
>   each reader per cycle.
> - **Before switching from one reader to the next, explicitly turn the RF
>   field off on the reader you're leaving and add a short settle delay.**
>   Leaving a field on while switching NSS to another chip on the same bus
>   causes crosstalk and unreliable reads.
> - Support two tag formats on each reader: ISO14443A (for reading Bambu
>   Lab spool RFID tags — Mifare Classic/compatible) and ISO15693 (for
>   ICODE SLIX2 tags, e.g. the OpenPrintTag open standard). Try one
>   protocol, then the other, each scan cycle.
> - If you use a third-party PN5180 ESP-IDF component/library, check its
>   deinit/teardown function carefully: it should only free the *shared*
>   SPI bus resources when explicitly told to (e.g. a `free_spi_bus`
>   parameter), not unconditionally — otherwise one reader failing to
>   initialize corrupts the bus for the other reader too. If the library
>   you find doesn't do this correctly, patch it or vendor a local copy
>   with the fix rather than working around it at the call site.
> - Add a watchdog so a wedged SPI transaction (e.g. from RF instability)
>   can't hang the scan loop forever with no recovery.
> - Run a small Wi-Fi web server with:
>   - A JSON API returning the current state of both readers (present/absent,
>     decoded tag fields, protocol used).
>   - A dashboard HTML page with **one column per physical reader board**,
>     each showing that reader's live status and decoded tag data, polling
>     the API every ~500ms.
> - Structure the reader count as a compile-time constant so the same
>   codebase can scale from 1 to 4 readers by changing a pin table and a
>   count, not by rewriting the scan loop.
>
> Ask me for my exact GPIO pin assignments before generating the pin
> table, and ask whether I want Bambu tag decoding, OpenPrintTag decoding,
> or both, if it's not already clear.

## After the firmware is written

- Flash it with `pio run -t upload` (PlatformIO) or `idf.py flash` (raw
  ESP-IDF), whichever your assistant set the project up with.
- Open the ESP32's IP address (check your router's DHCP client list, or
  its serial log on first boot) in a browser to confirm both reader
  columns update when you place a tag.
- If reads are unreliable on one board but not the other, check that
  board's antenna isn't shadowed by the other one, and that your power
  supply isn't marginal (a powered USB hub resolves most of these).
