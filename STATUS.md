# Status and change log

Part of the [Universal Screen protocol reference](README.md). One row per command or area; the tags are explained
in the README. "Captured" refers to the 2026-10-30 capture of L-Connect 3.

## LCD 1CBE:A088

| Opcode / area | Name | Layout | Captured | Open |
|---|---|---|---|---|
| transport | bulk `0x01` / `0x81`, 512-byte header + payload in one write | complete (PUBLISHED, FIRMWARE) | yes | bus speed (high speed INFERRED) |
| header | DES-CBC 504 bytes + `00` × 6 + `A1 1A` | complete; vectors in the LCD document | yes (decrypted) | |
| replies | plaintext, echo + `C8` + timestamp + body | complete (FIRMWARE); `C8` for every opcode | yes | whether a wrong-geometry JPEG is answered |
| `0x0A` | GetVer | complete | yes | |
| `0x0B` | Reboot | complete | yes (after `0x96`) | Reboot alone returns in LCD mode (FIRMWARE), not seen on the hardware |
| `0x0C` | Halt | no parameters; **do not send** | no | |
| `0x0D` | Rotate | complete: 3 / 1 / 2 / 0, rotator 180° / 0° / 180° / 0°, saved | yes | value 0 never sent by the vendor (INFERRED portrait flipped) |
| `0x0E` | Brightness | complete: 0..100, values above 0 saved | yes | why the vendor sends 50 at re-init |
| `0x0F` | FrameRate | complete: one byte, not saved, boot default 30 | yes (7) | effect on JPEG frames |
| `0x11` | GetH264Block | complete: not handled, chunk size always 202 752 | no | |
| `0x26`, `0x27`, `0x28`, `0x2A` | file create, write, one-shot write, delete | complete (FIRMWARE); **do not send** | no | |
| `0x2B`, `0x2C` | OTA, RebootApp | not handled by the application; **do not send** | no | |
| `0x33` | SetClock | complete: modes 0 off, 1 on (both saved), 2 time only; local time | yes | |
| `0x34` | StopClock | complete | yes | |
| `0x60`, `0x61`, `0x67`, `0x68`, `0x2E`, `0x76`, `0x77`, `0x78`, `0x80`, `0x97`, `0xCA` | other products' opcodes | not handled (FIRMWARE) | no | |
| `0x62` | GetFileSize | complete (FIRMWARE) | no | |
| `0x63` | QueryDir | complete (FIRMWARE): paged raw text, no reply header | no | |
| `0x64` | QueryStorage | complete: three LE KiB values; OBSERVED zeros without an SD card | no | |
| `0x65` | PushJpg | complete: 480 × 1920 baseline, ≤ 512 000 bytes | no (not exercised) | 4:2:2 / 4:4:4, progressive; maximum rate |
| `0x66` | PushPng | complete: overlay layer; clear with a transparent PNG | no (not exercised) | compositing over video |
| `0x6E`, `0x6F`, `0x70`, `0x71`, `0x72` | local H.264 / JPEG playback | complete (FIRMWARE) | no | never used by any host seen |
| `0x79` | StartPlay | complete: only `[8..11]` read by the firmware | yes | |
| `0x7A` | QueryBlock | complete | yes | |
| `0x7B` | StopPlay | complete | yes | |
| H.264 stream format | Annex-B 480 × 1920 4:2:0, no B-frames, SPS/PPS before IDRs | complete as far as tested; OBSERVED High profile 2026-11-02 | yes (vendor stream) | decoder level and bit-rate limits; single leading SPS/PPS |
| `0x7D` | SetConfig | complete (FIRMWARE); two fields of unknown meaning; avoid | no | |
| `0x96` | SwitchToDesktop | complete: creates the monitor-mode flag, Reboot must follow | yes | |
| `0xC9` | GetGroupPosition | complete: always zero | no | |
| `0xF0` | GetChipId | complete: 8 binary efuse bytes (= the USB serial) + 8 unknown bytes; OBSERVED 2026-10-31 | no | meaning of bytes 16..23 |
| `0xFD`, `0xFE` | factory test, test-flag reset | **do not send** | no | |
| monitor mode → LCD | HID output report, `5f3759df` + 504 zeros, no report ID on the wire | complete; OBSERVED 2026-10-30 and 2026-10-31 | yes | rev1 units (`1A86:ACE1`) |
| monitor-mode device | descriptors, identity query `AA 55 35 00`, vendor descriptor, EDID, configuration packet | complete for identification | yes | brightness, rotation and DDC/CI in monitor mode; records `06 00`, `05 64` |
| offline clock | drawn by the firmware about 10 s after the USB link drops | complete (FIRMWARE); OBSERVED during S3 2026-10-30 | yes (toggle) | date order, inactive backlight level, PC fully off |
| saved settings | brightness > 0, orientation, offline clock, preset | complete (FIRMWARE) | | |
| firmware update | | **deliberately not described** | no | |

Remaining UNKNOWN on the LCD: the bus speed, the reply to a wrong-geometry frame, the rate limits, the safety of
control commands during heavy H.264 ingest, the 8 bytes after the chip id, and the monitor-mode controls.

## Change log

- **2026-11-03, LED ring bench results.** Three ring items moved from INFERRED or UNKNOWN to OBSERVED with
  independently written code on the same unit: `0x11` produces no IN data (timed reads of 50 and 500 ms, while
  streaming and right after a single frame); each 20-LED packet takes effect on its own (lone offset-0, offset-20 and
  offset-40 packets each changed their chunk at once, so there is no end-of-frame signal); the ring accepted 206
  frames/s with no tearing (57, 111 and 206 delivered at 60, 120 and 240 requested, the three writes of a frame 1.0 to
  1.3 ms). The capture decode also shows L-Connect's cadence (20 to 22 frames/s) unchanged by its speed setting, so
  speed changes the animation step, not the interval. Ring sections 4.1, 7.3 and 13, the README's bench table and the
  ring rows above are updated. Host note added: a frame wait rounded to the Windows timer tick halves a 60 frames/s
  stream.
- **2026-11-02, first edition.** Written from one USBPcap capture of L-Connect 3 (2026-10-30, 05:17 to 05:29:
  the switch to monitor mode and back, the vendor's re-initialisation, the offline-clock toggle, the 180° button,
  the landscape/portrait switch), the .NET metadata of `lianli.lcd207.dll` 1.4.0.125 and `L-Connect.Core.dll`, a
  static read of the panel's application firmware `lianli88_0001_0023` (2026-10-30), bench sessions with
  independently written code (2026-10-27 to 2026-11-02), and the open-source projects credited in the README.
  New compared with what was public before: the panel's complete opcode switch and reply semantics (`C8` for
  everything), the meaning of the Rotate values and what the vendor sends for orientation, the offline-clock modes
  and the panel's offline behaviour, the saved settings, the storage and local-playback commands with their
  layouts, Halt, GetChipId and SetConfig on Lian Li firmware, how the monitor-mode switch works inside the panel
  (a flag file; Reboot alone does not leave LCD mode), the Windows framing of the switch-back report, the
  monitor-mode identity query, and this unit's monitor-mode descriptors.

## LED ring 0416:8050

| Command | Layout | Captured | Notes |
|---|---|---|---|
| `0x11` SetLeds | complete: `11 <first LED> 00 00` + 20 × RGB, three packets at offsets 0, 20, 40 = 60 LEDs | yes (2026-10-30, 45 285 packets = 15 095 frames, 20-22 frames/s); first published by lian-li-linux, OpenRGB and lianli-python-lcd-overlay; OBSERVED 2026-10-27 | bytes 2-3 always `00 00` on the wire, meaning UNKNOWN; per-packet latching, other offsets and the highest frame rate UNKNOWN (36 frames/s OBSERVED without errors) |
| `0x10` GetVersion | complete: request `10` + 63 × `00`, reply `10 <major> <minor>` + 61 × `00`; `10 01 03` = `lianliLed88_1_3` | yes (once, 05:19:14.878, reply 05:19:15.080); first published by lianli-python-lcd-overlay; OBSERVED 2026-10-27 | whether the vendor also sends it at service start was not seen |
| IN reply to `0x11` | none | yes (a read stayed pending for 10 min of frames and received nothing); OBSERVED 2026-11-03 (timed reads of 50 and 500 ms while streaming and right after a frame: nothing) | frames are fire-and-forget |
| per-packet latching | each 20-LED packet takes effect on its own; no end-of-frame signal | no (the vendor always sends all three) ; OBSERVED 2026-11-03 with lone offset-0, offset-20 and offset-40 packets | |
| streaming rate | at least 206 frames/s accepted (57, 111 and 206 delivered at 60, 120, 240 requested), writes 1.0-1.3 ms per frame | L-Connect: 20-22 frames/s regardless of its speed setting; OBSERVED 2026-11-03 for the fast rates | the ring's true ceiling |
| brightness, effect, speed, save, off | no such commands: all are host-side changes to the `0x11` frame contents | yes (every L-Connect change altered only the RGB bytes; brightness 0 % = all-zero frames) | L-Connect has no off control for the ring |
| vendor effects (19, `LightingMode` 0-18) | host-rendered frame streams; names and values METADATA, algorithms first published by lian-li-linux | Rainbow yes: 30 frames byte-identical to lian-li-linux's renderer | the other 18 not compared with the vendor; L-Connect's slider-to-level mapping UNKNOWN |
| `0x0E`, `0x12`, `0x15` | do not send | no | RF-dongle names and a possible reset in the ring's vendor enum (METADATA) |
| `0x20`-`0x23`, `0x25` | do not send | no | firmware erase, write, verify, reboot, info (METADATA); the update sequence is deliberately not described |
| any other first byte | do not send | no (none in the capture) | sibling Lian Li MCUs use such values for flash, NVRAM and boot-loader commands |
| descriptors | complete: bcdUSB 2.01, vendor class, bulk `0x01` / `0x81` 64 bytes, no serial string | yes (injected, 05:17:33.362); OBSERVED 2026-10-27 | full speed INFERRED |
| LED topology | complete for the observed unit: 7 LEDs per short side, 23 per long side; LED 0 on the bottom long side two LEDs left of its centre (landscape mount, seen from the front), indices counterclockwise; short-side centres at wire LEDs 17 and 47 | n/a | OBSERVED 2026-11-02 (single wire LEDs lit); supersedes the 2026-10-27 notes; other units UNKNOWN |
| standalone behaviour | last frame latched after the host exits, OBSERVED 2026-10-27; stays on the bus during the LCD's monitor mode, CAPTURED | partly | power-on state and S3 behaviour UNKNOWN |
