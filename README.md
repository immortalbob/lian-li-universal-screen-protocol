# Lian Li Universal Screen 8.8": USB protocol reference

An independent description of the USB protocols spoken by the Lian Li Universal Screen 8.8", a 480 × 1920
LCD panel with a 60-LED ring around it. Inside the enclosure a WCH USB hub (`1A86:8091`) joins **two USB
devices**, and the LCD has a second USB personality it switches to when it acts as a Windows monitor:

| Device | USB ID | What it does | Document |
|---|---|---|---|
| LCD panel, LCD mode | `1CBE:A088` | the 480 × 1920 panel driven by the host with DES-wrapped bulk commands: JPEG and PNG frames, H.264 playback decoded on the panel, brightness, orientation, the panel's own offline clock | [lianli-universal-screen-lcd.md](lianli-universal-screen-lcd.md) |
| LCD panel, monitor mode | `1A86:AD21` (rev2 units; rev1 units `1A86:ACE1`) | the same panel re-enumerated as a USB display (1920 × 480) for Lian Li's Windows display driver; one HID report switches it back to LCD mode | [lianli-universal-screen-lcd.md](lianli-universal-screen-lcd.md), section 9 |
| LED ring | `0416:8050` | the 60 LEDs (three groups of 20); colour frames streamed by the host over its own bulk pipe | [lianli-universal-screen-ledring.md](lianli-universal-screen-ledring.md) |

Coverage per command and change history: [STATUS.md](STATUS.md).

## Why this exists

The LCD's command header and its main frame and video commands were already public, from three independent
open-source projects (see Credits). What was missing: what Lian Li's own software (L-Connect 3) actually sends
for orientation, the offline clock and the switch to monitor mode and back; what the panel does with each
command, including the ones no host software uses; what it does by itself when the PC sleeps or is switched off;
and which commands are dangerous. On 2026-10-30 the USB traffic between L-Connect 3 and the panel was captured
with USBPcap while those functions were exercised, the panel's published application firmware was read
statically, and the results were checked by driving the same unit with independently written code
(2026-10-27 to 2026-11-02). This reference writes down what was seen, what the firmware says, and what is still
unknown.

## Scope

- The wire protocol of the LCD in both personalities: endpoints, the encrypted header, every opcode the
  panel's firmware handles, replies, sequences, timing and standalone behaviour. The LED ring has its own
  document.
- Not covered: L-Connect's user interface and host-side logic that never reaches the USB bus, the streaming
  protocol of monitor mode beyond what is needed to recognise and leave it, and the firmware-update procedure,
  which is deliberately **not** described beyond the warning not to trigger it.
- One unit was observed: a rev2 panel with LCD firmware `lianli88_0001_0023` (Lian Li's v1.5) and LED-ring
  firmware `lianliLed88_1_3`, with L-Connect 3 as the vendor host (its `lianli.lcd207.dll` at version
  1.4.0.125). Other firmware versions may differ.

## How to read the tags

Every statement carries one of these tags.

| Tag | Meaning |
|---|---|
| **CAPTURED** | seen on the USB bus in the capture of L-Connect 3 listed below; the time of day is given so the frame can be found in the raw file. LCD-mode headers were decrypted with the key in the LCD document |
| **METADATA** | read from the public .NET metadata tables of L-Connect 3's assemblies (`lianli.lcd207.dll`, `L-Connect.Core.dll`): enum, type, method and constant names and values, and user strings. Metadata says what the software *declares*, not what it sends |
| **FIRMWARE** | read statically from the panel's application firmware `lianli88_0001_0023` (see "Sources"). Firmware facts describe what the panel's code does with a request; they override earlier inferences where they differ |
| **PUBLISHED** | first published by an open-source project, which is named. Where a capture, the firmware or the bench re-checked the fact, the statement says so |
| **HOST** | read on the capturing PC without driving the device: Windows device properties of the unit (bus-reported names, compatible ids, interface GUIDs), the vendor's installed driver names, and the headers of the vendor's installed media files; used only to explain wire values |
| **OBSERVED** | seen while driving this unit with independently written third-party code (not the vendor software), with the date |
| **INFERRED** | a conclusion drawn from the evidence, not read from any source; the reasoning is given |
| **UNKNOWN** | a gap, with a note on what would close it |

## Captures and bench sessions

**The capture** (2026-10-30, 05:17 to 05:29, local time of the capturing PC, UTC-7). One USBPcap session of the
whole root hub with injected descriptors, filtered afterwards to the Lian Li devices. L-Connect 3 owned the panel;
the owner pressed one button at a time and noted the time.

| Time | What was exercised |
|---|---|
| 05:18:27 | L-Connect's "Desktop mode" button: LCD mode to monitor mode |
| 05:18:31 to 05:19:07 | the panel as a Windows monitor (descriptors, the identity query, the display driver's start-up) |
| 05:19:07.9 | L-Connect's "LCD mode" button: monitor mode back to LCD mode |
| 05:19:14.9 | L-Connect re-initialising the LCD and restarting its H.264 background |
| 05:19:47.9 and 05:19:56.5 | the offline-clock toggle, on then off |
| 05:20:02.8 and 05:20:05.8 | the "180" button, twice |
| 05:20:14.8 and 05:21:04.5 | landscape to portrait, and back to landscape |
| during the session | LED-ring effect, speed and brightness changes (see the LED-ring document) |

The raw capture file is not published (it holds unrelated traffic of the same root hub before filtering, and the
vendor's background video). The decoded frames quoted in these documents are complete for every command that
matters; video payloads are described, not reproduced.

**Bench sessions** (OBSERVED; independent third-party code driving the same unit, with L-Connect stopped):

| Date | What was checked |
|---|---|
| 2026-10-27 | GetVer, JPEG frames (filled and rotated content), brightness 30 and 80, a landscape dashboard at 10 fps, H.264 video at 30 fps, reopening after a lost handle; the LED ring's identification, GetVersion, frames, effects and an LED placement calibration |
| 2026-10-30 06:18 to 06:40 | Rotate 1 and 3, QueryStorage, GetChipId, two round trips to monitor mode and back, the offline clock during an S3 sleep and the return after the resume |
| 2026-10-31 | the raw GetChipId reply (17:45), another monitor-mode round trip, an S3 resume, a full power cycle (PSU off) |
| 2026-11-01 to 11-02 overnight | an S3 sleep of about 6.5 h: the LCD and the LED ring reopened by themselves after the resume |
| 2026-11-02 06:21 to 06:31 | H.264 streams from a different encoder (Windows Media Foundation, High profile), portrait and host-rotated landscape content |
| 2026-11-02 06:45 to 06:55 | the LED ring's layout, lighting single wire LEDs (0, 1, 47, 50, 51, 52, 55, 17) and noting where each appears |

## Sources

The facts in these documents come from these sources only:

1. A USB capture of the vendor software (L-Connect 3) driving the panel, made with USBPcap on the owner's own PC.
2. The public .NET metadata tables of L-Connect 3's assemblies, read with a small standard-library parser
   (no code loaded, no IL decoded).
3. **A static read of the panel's application firmware.** Lian Li distributes the panel's firmware for its own
   updater; the v1.5 application binary, `lianli88_0001_0023` (110 632 bytes, SHA-256
   `af6963ef359079e3cc30cc227b01d4aee44edc5343393b717f300407f2dd1809`, a MIPS32 ELF with its symbol table intact),
   was read statically on 2026-10-30 for interoperability. Nothing was executed, emulated or sent to the panel.
   No firmware code is reproduced here: the documents give only facts (opcode numbers, field layouts and
   behaviour) and, where they help, individual function or symbol names and file names.
4. The open-source projects listed under Credits, for the PUBLISHED statements.
5. For the OBSERVED statements: the panel's own replies and behaviour when driven by independently written
   third-party code on the same PC.

No vendor source code was used or seen. The only vendor strings reproduced are names from the metadata tables,
names and file names from the firmware's symbol and string tables, and strings that appear on the wire.

## Credits

Projects are linked to their repositories; the facts were taken from their source trees (default branch) as of
October 2026.

- **[lian-li-linux](https://github.com/sgtaziz/lian-li-linux)** (Linux daemon and drivers for Lian Li devices)
  first published the header builder for this panel family (`crates/lianli-devices/src/crypto.rs`: the DES key,
  the opcodes, the parameter layouts), the init sequence, the H.264 chunking and flow control, the close-and-reopen
  recovery, the 480 × 1920 / 512 000-byte frame limits (`crates/lianli-shared/src/screen.rs`), the desktop-mode
  device ids and switch-back magic (`crates/lianli-devices/src/display_switcher.rs`) and the TURZX monitor-mode
  streaming protocol (`crates/lianli-devices/src/turzx/`).
- **[hypercolor](https://github.com/hyperb1iss/hypercolor)** has a driver written for exactly this product
  (`crates/hypercolor-hal/src/drivers/lianli/universal_screen.rs`) with committed DES test vectors and a driver
  specification; it documented the reply status `0xC8`, the silent drop of wrong-geometry frames and the
  delivered frame rate.
- **[lianli-python-lcd-overlay](https://github.com/elniko/lianli-python-lcd-overlay)** (its `Linx` module, `Linx/linx.py`) documented the clock
  modes 0/1/2, the wake-from-desktop write and the PNG frame path.
- **[turing-smart-screen-python](https://github.com/mathoudebine/turing-smart-screen-python)** drives TURZX-branded
  panels (`library/lcd/lcd_comm_turing_usb.py`). Those run TURZX's own firmware, not Lian Li's build, so its facts
  apply only partly: the header, frame and H.264 commands agree, its storage-query and settings layouts turned out
  to match Lian Li's firmware, and two of its storage opcodes mean something else on this panel (LCD document,
  section 7).
- **[OpenRGB](https://gitlab.com/CalcProgrammer1/OpenRGB)** supports the LED ring and published its `0x11` frame layout independently.
- **Lian Li** is the vendor whose software and firmware were observed. Product and command names are Lian Li's.

Nothing from these projects (code, comments or prose) is copied here, only the protocol facts they established;
byte layouts, constants and the DES key/IV that they already publish are stated with credit.

## Licence

The documents in this folder are released under the **Creative Commons Attribution 4.0 International licence
(CC BY 4.0)**: use, copy, translate and build on them for any purpose, with credit to this work. See `LICENSE`.
No scripts are published with this reference.

## How to capture more

The capture was made with [USBPcap](https://desowin.org/usbpcap/) on Windows 11.

1. Install USBPcap. Run `USBPcapCMD.exe` once without arguments: it lists the root hubs (`\\.\USBPcap1`, ...) and
   their devices. Find the root hub that carries `1A86:8091` (the hub inside the screen) with `1CBE:A088` and
   `0416:8050` below it.
2. **Capture the whole root hub**, not a device list: the panel changes its address every time it switches
   personality (`1CBE:A088` → `1A86:AD21` → `1CBE:A088`), so a device filter loses it. From an elevated prompt:

   ```
   USBPcapCMD.exe -d \\.\USBPcap1 -A --inject-descriptors -o screen-raw.pcap
   ```

   `-A` captures every device on the hub; without it and without `--devices` USBPcapCMD exits at once. Stop
   with Ctrl+C (a script that sends Ctrl+C to USBPcapCMD should ignore it itself).
3. Afterwards keep only the Lian Li devices: find every address they used from the device descriptors in the file
   (`1CBE:A088`, `1A86:AD21` or `1A86:ACE1`, `1A86:8091`, `0416:8050`) and drop everything else. The root hub
   usually also carries your keyboard and mouse, so delete the raw file once it is filtered. After a sleep or a
   reboot the injected descriptor list can carry stale addresses; check which address actually carries the
   traffic you expect.
4. Do one thing at a time in L-Connect and write down the clock time of each action. L-Connect streams its
   background video (and the ring's colour frames) continuously, so the interesting commands are found by
   setting the `0x79` / `0x7A` stream and the ring frames aside.
5. To read LCD-mode commands, take the first 512 bytes of each bulk OUT transfer on endpoint `0x01`, check the
   trailer `A1 1A` at 510, and DES-CBC-decrypt bytes 0..503 with the key in the LCD document; byte 0 of the
   plaintext is the opcode. Replies on `0x81` are plaintext. Each USBPcap record starts with a header
   `u16 headerLength, u64 irpId, u32 status, u16 function, u8 info, u16 bus, u16 device, u8 endpoint,
   u8 transferType, u32 dataLength`; transfer type 3 is bulk, 1 interrupt, 2 control.
6. Monitor mode streams the whole desktop as video over the same hub: the raw file grows by about 20 MB per minute,
   so do not linger there.

Never replay captured packets to the device, and never trigger a firmware update to capture it.

Still uncaptured, in case you want to contribute: anything L-Connect does with the panel's storage, the
"secondary screen" setting, the monitor-mode backlight and rotation requests, and a rev1 unit (`1A86:ACE1`). See
[STATUS.md](STATUS.md).

## Safety

The panel runs Linux and its command set includes file writes, file deletion, a halt, a factory test and the
monitor-mode switch. The LCD document lists those commands with a do-not-send warning (section 12). Apart from
the OBSERVED statements, nothing in this reference was tested beyond observing the vendor software; anyone sending
commands to their own panel does so at their own risk. In particular, the panel answers **every** command with
the same "received" status, whether or not it understands it, so probing unknown opcodes cannot tell you anything
and may do something you cannot undo.
