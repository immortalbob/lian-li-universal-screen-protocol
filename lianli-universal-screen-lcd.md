# Lian Li Universal Screen 8.8": LCD protocol (USB `1CBE:A088`)

Part of the [Universal Screen protocol reference](README.md). Tags (**CAPTURED**, **METADATA**, **FIRMWARE**,
**PUBLISHED**, **OBSERVED**, **INFERRED**, **UNKNOWN**) and the capture list are explained there. Capture times
are 2026-10-30, local time (UTC-7). Unless a field says otherwise, sizes and lengths inside a command are
**big-endian** and the header timestamp is **little-endian**. Opcode names are Lian Li's where a vendor name
exists: from the `CmdType` enum of `lianli.lcd207.dll` (METADATA) or from the firmware's function names
(FIRMWARE); otherwise the name used by lian-li-linux. An OBSERVED statement without its own date was seen
throughout the bench sessions listed in the README (2026-10-27 to 2026-11-02).

## 1. Device and transport

### 1.1 Identity

| Item | Value | Status |
|---|---|---|
| USB ID | `1CBE:A088`, `bcdDevice` `0000` | HOST |
| Class | vendor (`FF/00/00`), one interface (0), not composite, no HID | PUBLISHED (hypercolor); OBSERVED (Windows lists one interface) |
| Endpoints | bulk OUT `0x01`, bulk IN `0x81`, 512-byte max packet | PUBLISHED (hypercolor, lian-li-linux); OBSERVED in use |
| Windows driver | the device advertises the Microsoft OS compatible ID `MS_COMP_WINUSB`, so Windows binds WinUSB with no INF | HOST |
| Product string | `8.8" Universal Screen-1.5` | HOST (as reported by the bus to Windows) |
| Serial number | 16 hex digits; they are the first 8 bytes of the SoC's efuse chip id (section 4.8), written as `<serial>` in this reference | HOST; the chip-id match OBSERVED 2026-10-31 |
| Parent | WCH hub `1A86:8091` inside the enclosure; the LED ring `0416:8050` is its sibling | HOST |
| Firmware | GetVer answers `lianli88_0001_0023` | CAPTURED (05:19:14.9); OBSERVED |
| Bus speed | UNKNOWN; a 512-byte bulk max packet is only legal at high speed, so INFERRED high speed | |
| Monitor-mode identity | `1A86:AD21` on rev2 units, `1A86:ACE1` on rev1 units (section 9) | CAPTURED (AD21); PUBLISHED (ACE1, lian-li-linux) |

The HOST rows of this table are the unit's Windows device properties (USB ids, driver binding, product string,
device tree), read before and during the bench sessions.

The LCD is a **TURZX** design, the same silicon family as the TURZX-branded "Turing" smart screens, but it runs
Lian Li's own firmware build. The TURZX-branded 8.8" enumerates as `1CBE:0088` (PUBLISHED, turing-smart-screen-python).

### 1.2 Inside the panel (FIRMWARE)

- The panel is an Ingenic XBurst2 SoC running Linux 3.10. The command handler is a user-space application that
  reads requests from `/dev/ttyGS0`, a Linux USB *generic serial* gadget set up in vendor-class mode. That gadget is
  what the host sees as `1CBE:A088` with two bulk endpoints.
- The application polls the USB device controller's state once a second and treats `configured` as "a host is
  present" (section 10).
- JPEG and H.264 are decoded in hardware. Rotation is the SoC framebuffer rotator. Brightness is a PWM backlight.
- In monitor mode the boot scripts start a different USB gadget (the TURZX display gadget) instead of the serial
  one (section 9.4).

### 1.3 Transport

- Host → panel: one **bulk OUT** write on `0x01` per command, the 512-byte header followed, in the same write, by
  the command's payload if it has one (PUBLISHED, lian-li-linux and hypercolor; CAPTURED: L-Connect's frames and
  chunks are single transfers of header plus payload; FIRMWARE: the handler reads exactly 512 bytes, then reads the
  stated payload length).
- Panel → host: one **bulk IN** reply on `0x81` per command (section 2.5).
- There is **no report ID**. Byte 0 of a bulk OUT transfer is the first ciphertext byte of the header.
- No keep-alive is needed (PUBLISHED, lian-li-linux; FIRMWARE: no idle timeout on host traffic, section 10).
- Do not send `SET_INTERFACE`: lian-li-linux avoids it because related Lian Li devices drop the next OUT transfer
  after it (PUBLISHED). Not tried on this panel.

### 1.4 Exclusivity

The panel's interface can be held by one process at a time (WinUSB). L-Connect 3's service opens it whenever it runs
and keeps it open, so a third-party driver must have the vendor service stopped first (OBSERVED: the open fails while
the service runs). The same hub carries the LED ring, so a USB port reset or hub reset of the panel also disturbs the
ring; recover by closing and reopening the handle instead (section 11).

## 2. Command header

Every command is one 512-byte header: a 500-byte plaintext, encrypted, then a fixed trailer. PUBLISHED by
lian-li-linux and hypercolor (and lianli-python-lcd-overlay); CAPTURED (every L-Connect header in the capture
decrypts to this layout); FIRMWARE (the handler checks exactly these fields).

### 2.1 Plaintext (500 bytes)

| Offset | Size | Field | Value |
|---|---|---|---|
| 0 | 1 | opcode | section 3 |
| 1 | 1 | reserved | `00` |
| 2 | 2 | magic | `1A 6D` (checked by the firmware) |
| 4 | 4 | timestamp, u32 **little-endian** | milliseconds; see 2.3 |
| 8 | 492 | parameters | per opcode, zero padded |

Parameters longer than 492 bytes do not fit; payload data travels after the header (2.4).

### 2.2 Encryption and wire layout

**DES-CBC**, key = IV = ASCII `slv3tuzx` (`73 6C 76 33 74 75 7A 78`), **PKCS#7** padding: the 500 plaintext bytes
become 504 ciphertext bytes (the padding block ends in four `04` bytes). PUBLISHED (lian-li-linux, hypercolor);
FIRMWARE (the key and IV are both in the firmware's read-only data, and it decrypts with them).

| Offset | Size | Field | Value |
|---|---|---|---|
| 0 | 504 | ciphertext | DES-CBC(plaintext ∥ PKCS#7 padding) |
| 504 | 6 | reserved | `00 00 00 00 00 00` |
| 510 | 2 | trailer | `A1 1A` |

FIRMWARE: a request is accepted only when `[510] = A1` and `[511] = 1A`; then bytes 0..503 are decrypted and the
plaintext must carry `1A 6D` at 2..3. A packet failing either check is not processed as a command (INFERRED: the
handler skips it; what it replies then is UNKNOWN).

The key is obfuscation, not security: it ships with the vendor software and in public code. The DES key is neither
weak nor semi-weak, so standard libraries accept it (for example .NET's `DES` class).

turing-smart-screen-python zero-pads the plaintext to 504 bytes instead of using PKCS#7; only the last cipher block
differs (PUBLISHED, on TURZX firmware). INFERRED: the firmware never looks at the padding block, so both work; use
PKCS#7 as the vendor does.

### 2.3 Timestamp

FIRMWARE: the timestamp is **never checked**, only copied into the reply (2.5). The open-source drivers send
milliseconds since their session started, made strictly increasing (PUBLISHED, lian-li-linux and hypercolor);
turing-smart-screen-python sends milliseconds since local midnight. Any value works; a strictly increasing one costs
nothing and makes replies easy to match.

### 2.4 Payload

Commands that carry data (frames, video chunks, file data) state the payload length in their parameters, and the
payload follows the header **in the same bulk write, at its natural length, with no padding** (PUBLISHED; CAPTURED;
FIRMWARE: the handler reads the stated number of bytes after the header).

### 2.5 Replies

FIRMWARE: **every** request, handled or not, gets one reply of 512 bytes on `0x81`, **in plaintext**:

| Offset | Size | Field |
|---|---|---|
| 0 | 1 | opcode echo |
| 1 | 1 | status, always `0xC8` |
| 2 | 4 | the request's timestamp bytes (plaintext 4..7) |
| 6 | 2 | zero |
| 8 | 504 | body, zero unless the handler fills it |

So `0xC8` means "received", not "supported" or "succeeded": an opcode the firmware does not know gets the same reply
with an empty body. A driver cannot discover support by probing, and should not try.

PUBLISHED (hypercolor, lian-li-linux): `0xC8` is the success code; lian-li-linux drains the IN pipe after every reply
with short reads so a stray reply cannot shift the next one. Recommended.

Exceptions and timing:

- The QueryDir pages (section 7) are raw text with no echo or status (FIRMWARE).
- hypercolor reports that a frame with the wrong geometry is never answered (PUBLISHED). The firmware read suggests
  every request is answered; which is right for a wrong-geometry JPEG is UNKNOWN (section 14).
- After the panel's offline clock has been running, the first command can go unanswered for more than 2 s while the
  clock thread winds down (OBSERVED 2026-10-30 06:41, GetVer after a resume; on 2026-10-31 a GetVer after a resume
  was answered at once).

## 3. Opcode table

"Firmware" says whether the panel's application handles the opcode (FIRMWARE: its handler is a chain of 31
comparisons; anything else takes the default, empty reply). "Vendor" says whether L-Connect was seen sending it in
the capture ("not seen" means not among the decoded commands of that one session; "not exercised" means the session
never used the feature that would send it). "Section" points to the details.

| Opcode | Name | Firmware | Vendor | Status | Section |
|---|---|---|---|---|---|
| `0x0A` | GetVer | handled | CAPTURED | complete | 4.1 |
| `0x0B` | Reboot | handled | CAPTURED (after `0x96`) | complete; **mode switch only** | 4.2, 9 |
| `0x0C` | Halt | handled | not seen | **do not send** | 4.2, 12 |
| `0x0D` | Rotate | handled | CAPTURED | complete | 4.3 |
| `0x0E` | Brightness | handled | CAPTURED | complete | 4.4 |
| `0x0F` | FrameRate | handled | CAPTURED | complete; effect on JPEG UNKNOWN | 4.5 |
| `0x11` | GetH264Block | **not handled** (always an empty body) | not seen | complete (the answer is always "use the default") | 6.1 |
| `0x26` | CreateFile | handled | not seen | FIRMWARE only; **do not send** | 7 |
| `0x27` | WriteFileChunk | handled | not seen | FIRMWARE only; **do not send** | 7 |
| `0x28` | UpdateFireWare (vendor name; a one-shot file write) | handled | not seen | FIRMWARE, PUBLISHED; **do not send** | 7 |
| `0x2A` | DelFile | handled | not seen | FIRMWARE only; **do not send** | 7 |
| `0x2B` | OTA | not handled by this application | not seen | METADATA; **do not send** | 12 |
| `0x2C` | RebootApp | not handled by this application | not seen | METADATA; **do not send** | 12 |
| `0x2E` | WarnSwitch | not handled | not seen | another product's command | 3.1 |
| `0x33` | SetClock | handled | CAPTURED | complete | 4.6 |
| `0x34` | StopClock | handled | CAPTURED | complete | 4.6 |
| `0x60` | GetTemperature | not handled | not seen | another product's command | 3.1 |
| `0x61` | SetPumpSpeed | not handled | not seen | another product's command; **do not send** | 3.1 |
| `0x62` | GetFileSize (firmware); GetPumpSpeed (vendor enum) | handled as GetFileSize | not seen | FIRMWARE only | 7 |
| `0x63` | QueryDir | handled | not seen | FIRMWARE only (paged text) | 7 |
| `0x64` | QueryStorage | handled | not seen | complete; OBSERVED | 7 |
| `0x65` | PushJpg | handled | not exercised | complete | 5.1 |
| `0x66` | PushPng | handled | not exercised | complete | 5.2 |
| `0x67` | ClearPng | not handled | not seen | another product's command | 5.2 |
| `0x68` | HideShowFrames | not handled | not seen | another product's command | 3.1 |
| `0x6E` | PlayLocalH264 | handled | not seen | FIRMWARE only | 7 |
| `0x6F` | StopLocalH264 | handled | not seen | FIRMWARE only | 7 |
| `0x70` | GetLocalH264Playing | handled | not seen | FIRMWARE only | 7 |
| `0x71` | ShowLocalJpg | handled | not seen | FIRMWARE only | 7 |
| `0x72` | GetImgPlaying | handled | not seen | FIRMWARE only | 7 |
| `0x76` | GetH264Frame | not handled | not seen | METADATA only | 3.1 |
| `0x77`, `0x78` | StartPlay1, StartPlay2 | not handled | not seen | slots 1 and 2 of other products | 6.2 |
| `0x79` | StartPlay | handled | CAPTURED | complete | 6.2 |
| `0x7A` | QueryBlock | handled | CAPTURED | complete | 6.3 |
| `0x7B` | StopPlay | handled | CAPTURED | complete | 6.4 |
| `0x7D` | SetConfig | handled | not seen | FIRMWARE; PUBLISHED (turing, on TURZX firmware); avoid | 4.9 |
| `0x80` | LcdRevision | not handled | not seen | other products only | 3.1 |
| `0x96` | SwitchToDesktop | handled | CAPTURED | complete | 9 |
| `0x97` | Test | not handled | not seen | METADATA only; factory | 12 |
| `0xC9` | GetGroupPosition (firmware); GetPosIndex (vendor enum) | handled | not seen | FIRMWARE: always zero | 4.8 |
| `0xCA` | SetScreenOffset | not handled | not seen | METADATA only | 3.1 |
| `0xF0` | GetChipId | handled | not seen | complete; OBSERVED | 4.8 |
| `0xFD` | TestH264 (firmware); FactoryH264Test (vendor enum) | handled | not seen | **do not send** (factory test) | 12 |
| `0xFE` | (clears a test flag) | handled | not seen | FIRMWARE only; no reason to send | 12 |

### 3.1 Opcodes for other Lian Li products

lcd207's `CmdType` (METADATA) and lian-li-linux serve a whole family of Lian Li LCDs with this header (AIO pump
heads, the HydroShift II OLED, case panels), so some opcodes belong to other products:

- `0x60` / `0x61` / `0x62` are temperature and pump commands for AIO pump-head LCDs (METADATA names). This panel
  has no pump. The firmware does not handle `0x60` or `0x61`, and `0x62` is **GetFileSize** here (FIRMWARE).
- `0x67` ClearPng, `0x68` HideShowFrames and `0x2E` WarnSwitch are sent by lian-li-linux only to the HydroShift II
  OLED (PUBLISHED); in lcd207 they appear in that product's nested enum (METADATA). Not handled here (FIRMWARE).
- `0x80` LcdRevision is a revision probe lian-li-linux sends to other panels (PUBLISHED). Not handled here
  (FIRMWARE): its reply body is zero, which those drivers read as "legacy revision".
- `0x76` GetH264Frame, `0xCA` SetScreenOffset and `0x97` Test are names in `CmdType` (METADATA) with no handler in
  this firmware.

turing-smart-screen-python's TURZX-firmware opcodes overlap partly (PUBLISHED, re-checked against FIRMWARE):
its storage query `0x64` and its settings command `0x7D` have the same layouts on Lian Li's firmware; its "open"
and "write" (`0x26`, `0x27`) match CreateFile and WriteFileChunk; its pre-video commands `0x6F` and `0x70` are
StopLocalH264 and GetLocalH264Playing here; but its "delete" uses `0x28`, which is a **file write** here, and its
"play file" uses `0x62`, which is GetFileSize here. Never port TURZX storage commands to this panel without
checking this table.

## 4. Device commands

### 4.1 GetVer (`0x0A`)

No parameters. Reply body: ASCII firmware name at `[8..]`, zero terminated (FIRMWARE: the string is compiled into the
application). CAPTURED and OBSERVED: `lianli88_0001_0023`.

lian-li-linux's strict check (reply byte 0 = `0x0A`, non-empty, printable ASCII) is a good validity rule (PUBLISHED).
The name format is `lianli88_<4 digits>_<4 digits>`; another unit reported `lianli88_0001_0018` (PUBLISHED, via
hypercolor).

### 4.2 Reboot (`0x0B`) and Halt (`0x0C`)

- **Reboot** `0x0B`, no parameters: the firmware runs a Linux `reboot` (FIRMWARE). The panel leaves the bus and
  comes back. Whether it comes back in LCD or monitor mode depends only on the flag file that `0x96` creates
  (section 9.4): **Reboot alone returns in LCD mode** (FIRMWARE). lianli-python-lcd-overlay and hypercolor state
  that Reboot alone enters desktop mode (PUBLISHED); the firmware contradicts that, and L-Connect sent `0x96`
  first (CAPTURED). A plain reboot was not tried on the bench (UNKNOWN on the hardware). L-Connect's Reboot was
  answered `C8` before the panel left (CAPTURED); a driver should still not depend on that reply.
- **Halt** `0x0C`, no parameters: the firmware runs a Linux `halt` (FIRMWARE). The panel's Linux shuts down and it
  stays dark and off the bus until its power is cut (INFERRED). Never send.

### 4.3 Rotate (`0x0D`)

Parameter `[8]` = orientation value 0..3. This is **not** a number of quarter turns, although lian-li-linux and
lianli-python-lcd-overlay build it as `rotation & 3` (PUBLISHED; neither sends it to this panel).

FIRMWARE: the handler maps the value to the SoC framebuffer rotator and to an orientation hint for the
firmware's own drawings (the offline clock and PNG overlays):

| Value | Rotator | Hint | What it looks like |
|---|---|---|---|
| `3` | 180° | landscape | L-Connect's normal landscape (CAPTURED: sent after a switch to landscape) |
| `1` | 0° | landscape | the same picture turned 180° (CAPTURED: L-Connect's "180" button sends 1, and 3 again on the second press; OBSERVED 2026-10-30: third-party Rotate 1 flipped the picture 180°, Rotate 3 restored it) |
| `2` | 180° | portrait | L-Connect's portrait (CAPTURED) |
| `0` | 0° | portrait | INFERRED: portrait turned 180°; never sent by the vendor |

Both the value and the rotator angle are saved in the panel's configuration file and **survive power cycles**
(FIRMWARE). The rotator only ever turns by 0° or 180°: the panel is a 480 × 1920 portrait buffer whatever the
value, and landscape content is drawn rotated by the host into that buffer (section 5.3). A driver can therefore use
1 ↔ 3 (or 0 ↔ 2) for a 180° flip without touching its frames, and the flip takes effect on a running H.264 stream
(CAPTURED: L-Connect left its stream running; OBSERVED 2026-10-30 on a dashboard of JPEG frames).

Captured examples (plaintext, `tt` = timestamp):

```
05:20:02.803  0D 00 1A 6D tt tt tt tt  01 00 ...     "180" pressed (flip)
05:20:05.755  0D 00 1A 6D tt tt tt tt  03 00 ...     "180" pressed again (normal landscape)
```

A landscape/portrait change is heavier in L-Connect; see section 8.3. A driver that renders its own content can
leave the panel at 3 and rotate on the host, as lian-li-linux and hypercolor do (they never send `0x0D`).

### 4.4 Brightness (`0x0E`)

Parameter `[8]` = 0..100 (percent). PUBLISHED (lian-li-linux: raw 0..100 for this panel, clamped to 100);
METADATA (L-Connect's constants `ScreenBrightnessMin = 0`, `ScreenBrightnessMax = 100`); CAPTURED; OBSERVED
2026-10-27 (30 and 80 applied and shown).

FIRMWARE: written to the PWM backlight. A value above 0 is **saved** and applied at the next boot; **0 is applied
but not saved**, so a panel turned off with brightness 0 comes back lit after a power cycle.

CAPTURED: L-Connect's re-initialisation sent `[8] = 0x32` (50) while its slider showed 100 % (05:19:14.9); the reason
is UNKNOWN.

There is no screen on/off command; brightness 0 is the off switch (PUBLISHED, every reference does this).

### 4.5 FrameRate (`0x0F`)

Parameter `[8]` = frames per second, one byte. FIRMWARE: sets the playback frame time; **not saved**; the boot
default is 30. CAPTURED: L-Connect sent 7 before its background stream. PUBLISHED: lian-li-linux sends 120 at init
and the clip's rate before each stream; turing-smart-screen-python sends 25. METADATA: L-Connect renders this panel
at `ScreenFrameRate = 24`. OBSERVED: 30 before a 30 fps clip plays it smoothly (2026-10-27, 2026-11-02).

INFERRED: the value paces H.264 playback. Whether it affects how fast JPEG frames are shown is UNKNOWN.

### 4.6 SetClock (`0x33`) and StopClock (`0x34`)

**SetClock** parameters:

| Offset | Field |
|---|---|
| 8..9 | year, u16 big-endian |
| 10 | month 1..12 |
| 11 | day |
| 12 | hour |
| 13 | minute |
| 14 | second |
| 15 | mode |

PUBLISHED (lian-li-linux, hypercolor, lianli-python-lcd-overlay); CAPTURED; FIRMWARE.

**Send local wall-clock time.** FIRMWARE: the handler sets the panel's system clock from these fields as if they
were UTC and prints the clock with the same convention, so what you send is what it shows. CAPTURED: L-Connect sent
its local time.

| Mode | Meaning | Status |
|---|---|---|
| `0` | offline clock **off** (saved) and the clock set | CAPTURED (L-Connect's toggle off, 05:19:56.5); FIRMWARE |
| `1` | offline clock **on** (saved) and the clock set | CAPTURED (toggle on, 05:19:47.9); FIRMWARE; OBSERVED 2026-10-30 |
| `2` | set the clock only; the saved on/off choice is left alone | CAPTURED (every re-initialisation); FIRMWARE (any value ≥ 2 acts like 2) |

The **offline clock** is a clock the firmware draws by itself after the host has gone (section 10.2). It is never
drawn while a host is attached, which is why toggling it in L-Connect changes nothing on the panel at the time
(CAPTURED: the owner saw no change; FIRMWARE). The choice is saved in the panel's configuration file and survives
power cycles (FIRMWARE). METADATA: L-Connect's request types `GetOfflineModeClock = 22` and
`SetOfflineModeClock = 23`.

Captured examples:

```
05:19:47.888  33 00 1A 6D tt tt tt tt  07 EA 0A 1E 05 13 2F 01   2026-10-30 05:19:47, mode 1 (on)
05:19:56.541  33 00 1A 6D tt tt tt tt  07 EA 0A 1E 05 13 38 00   2026-10-30 05:19:56, mode 0 (off)
```

L-Connect sent each toggle as a single SetClock, with no StopClock around it, while its H.264 stream kept running
(CAPTURED).

**StopClock** `0x34`, no parameters (lian-li-linux sends `[8] = 0`; CAPTURED the same). FIRMWARE: asks the offline-
clock thread to stop if it is running. Every L-Connect re-initialisation sends SetClock mode 2 followed by StopClock
(CAPTURED), and so do the open-source drivers (PUBLISHED).

### 4.7 Commands that change what the firmware accepts

FIRMWARE: **any** received command stops a running offline clock. While the clock thread is still winding down,
PushJpg, PushPng and StartPlay are **ignored** (they are accepted only while the clock is idle). The clock thread
then restores the rotation. So the first frames after an offline-clock period can be lost. INFERRED: the
StopPlay → SetClock(2) → StopClock preamble every driver sends gives the thread time to exit; a short pause after
StopClock (a few hundred milliseconds) and a retry of the first frame are cheap safeguards. OBSERVED 2026-10-30
06:41: after an S3 sleep during which the clock ran, the first frame write timed out; after a reopen the next ones
were shown.

### 4.8 GetChipId (`0xF0`) and GetGroupPosition (`0xC9`)

- **GetChipId** `0xF0`, no parameters. Reply body: `[8..15]` = the 8-byte efuse chip id of the SoC, **binary**;
  `[16..23]` = 8 more bytes of unknown meaning (FIRMWARE: the handler copies a 16-byte buffer read from the efuse at
  boot). OBSERVED 2026-10-31 17:45: bytes 8..15 written as hex are exactly the USB serial number (`<serial>`); bytes
  16..23 were non-zero (`<8 more bytes>`). The monitor-mode identity query returns the same 16 bytes (section 9.2).
  A safe read-only query.
- **GetGroupPosition** `0xC9`, no parameters. Reply `[8]` = group, `[9]` = position (FIRMWARE). Both are never set
  in this firmware, so the reply is always zero. The vendor enum calls the opcode `GetPosIndex` (METADATA).

### 4.9 SetConfig (`0x7D`)

Parameters (FIRMWARE; the same layout as turing-smart-screen-python's "save settings" on TURZX firmware, PUBLISHED):

| Offset | Field |
|---|---|
| 8 | brightness |
| 9 | "mode": what to show when a host attaches but sends nothing (1 = the local image player, 2 = a local H.264 file, other = nothing; section 10.1) |
| 10 | UNKNOWN (turing: reserved) |
| 11 | orientation, as Rotate (4.3) |
| 12 | UNKNOWN (turing: sleep timeout) |
| 13 | offline clock on/off |

FIRMWARE: all of them are written to the configuration file at once. L-Connect never sent it in the capture, and
everything it sets is reachable through Brightness, Rotate and SetClock. Avoid it: a wrong value in a saved field
survives power cycles.

## 5. Frames

### 5.1 PushJpg (`0x65`)

Parameters `[8..11]` = JPEG length (BE); the JPEG follows the header in the same write (2.4). Reply body `[8]` =
number of buffered frames (PUBLISHED, lian-li-linux).

| Rule | Status |
|---|---|
| geometry **480 × 1920** (portrait); anything else is dropped silently and the previous picture stays | PUBLISHED (hypercolor, lian-li-linux); OBSERVED: 480 × 1920 frames are shown |
| baseline (SOF0) JPEG, 8-bit; 4:2:0 subsampling is the safe choice | PUBLISHED (every reference sends baseline, hypercolor 4:2:0; lian-li-linux at quality 95); OBSERVED 2026-10-27 onwards (baseline JPEGs) |
| 4:2:2 and 4:4:4 | UNKNOWN on this firmware (turing-smart-screen-python uses them on TURZX firmware) |
| progressive or arithmetic-coded JPEG | UNKNOWN; assume rejected |
| at most **512 000 bytes** | PUBLISHED (lian-li-linux, hypercolor); larger frames are refused host-side by both |

FIRMWARE: the handler first stops a local H.264 playback if one runs, then decodes with the hardware JPEG decoder
straight to the framebuffer. It is ignored while the offline clock winds down (4.7).

lianli-python-lcd-overlay reports that PushJpg goes unanswered for images over about 2 KB under libusb on Linux, and
uses PushPng for everything (PUBLISHED); lian-li-linux and hypercolor use PushJpg without that problem, and so did
the bench (OBSERVED). UNKNOWN cause.

### 5.2 PushPng (`0x66`) and the overlay

Parameters `[8..11]` = PNG length (BE); the PNG follows. PUBLISHED (lian-li-linux, lianli-python-lcd-overlay).
FIRMWARE: decoded as an **overlay** above the JPEG or video background, oriented by the orientation hint of 4.3;
ignored while the offline clock winds down (4.7).

- Clear the overlay by pushing a fully transparent 480 × 1920 PNG; that is what every reference does for this panel
  (PUBLISHED) and what the independent driver did at every start (OBSERVED). ClearPng `0x67` is not handled by this
  firmware (FIRMWARE).
- lianli-python-lcd-overlay reports PNGs accepted at any size (PUBLISHED); all OBSERVED pushes were 480 × 1920.
- Whether an overlay composites over a playing H.264 stream: INFERRED yes (the vendor's templates draw over video,
  METADATA method names `ApplyTemplate`, `PushPng`); not checked on the bench (UNKNOWN).
- Same 512 000-byte cap as JPEG (PUBLISHED).

### 5.3 Geometry and landscape content

The panel takes only 480 × 1920 frames (5.1), and its rotator only turns by 180° (4.3). Landscape content is drawn
by the host rotated a quarter turn inside the 480 × 1920 buffer (PUBLISHED, lian-li-linux and hypercolor;
OBSERVED 2026-10-27 and since: landscape dashboards and host-rotated landscape video shown upright). METADATA:
L-Connect's constants `ScreenLandscapeWidth/Height = 1920/480`, `ScreenPortraitWidth/Height = 480/1920`.

### 5.4 Frame flow control

PUBLISHED (lian-li-linux): when the PushJpg reply's `[8]` is above 3, poll QueryBlock (`0x7A`) every 50 ms until
`[8]` is 2 or less before the next frame. OBSERVED: 10 fps dashboards ran for days with this rule. The
highest sustainable frame rate is UNKNOWN: lian-li-linux declares 120, hypercolor delivers 30, L-Connect renders at
24 (METADATA) and streams at 7 (CAPTURED).

## 6. H.264 playback (decoded on the panel)

The host streams an Annex-B H.264 elementary stream in chunks; the panel buffers and decodes them in hardware
(PUBLISHED; CAPTURED: L-Connect's background is such a stream; FIRMWARE: a ring of blocks drained by a playback
thread).

### 6.1 GetH264Block (`0x11`)

lian-li-linux and turing-smart-screen-python ask the chunk size with `0x11` (reply `[8..11]` u32 BE, 0 = use the
default 202 752 bytes; PUBLISHED). **This firmware does not handle `0x11`** (FIRMWARE), so the reply body is zero and
the chunk size is always the default **202 752** bytes (`0x31800`). CAPTURED: L-Connect's chunk transfers were up to
203 264 bytes, which is 512 + 202 752 (INFERRED: header plus a full chunk). Sending `0x11` is harmless; skipping it
is equivalent.

### 6.2 StartPlay (`0x79`)

| Offset | Field | Value |
|---|---|---|
| 8..11 | chunk length, u32 BE | 1..1 048 576; normally 202 752 except the last chunk |
| 12 | last-chunk flag | 1 on the final chunk of a non-looping clip, else 0 |
| 13 | play count | 0 |
| 14..17 | play tick, u32 BE | 0 |

The chunk follows the header in the same write. PUBLISHED (lian-li-linux; lianli-python-lcd-overlay puts 1 in
`[13]`, lian-li-linux 0, and both work); OBSERVED with 0. FIRMWARE: the handler reads only the length at 8..11
(INFERRED: bytes 12..17 are ignored by this panel). Reply `[8]` = blocks buffered (FIRMWARE, PUBLISHED). Ignored while
the offline clock winds down (4.7).

`0x77` and `0x78` (StartPlay for slots 1 and 2 on other products, reply levels in `[9]` and `[10]`) are not handled
here (FIRMWARE).

### 6.3 QueryBlock (`0x7A`) and flow control

No parameters. Reply `[8]` = blocks buffered (FIRMWARE, PUBLISHED). CAPTURED: L-Connect interleaves QueryBlock with
its chunks.

Flow control (PUBLISHED, lian-li-linux, matched by the other references; OBSERVED):

1. After each StartPlay, read the reply's `[8]`.
2. If it is above 3, poll QueryBlock every 50 ms until it is 2 or less (lian-li-linux gives up after 30 s).
3. If a chunk's reply is missing, send one QueryBlock instead; if that is unanswered too, stop and recover (11).
4. Pace chunks at no less than max(1 / fps, 30 ms) apart.
5. To loop, start the file again from its first byte and never set the last-chunk flag.
6. For a finite clip, set the flag on the last chunk and do **not** send StopPlay afterwards, so the panel plays out
   its buffer (PUBLISHED).

If the host stops feeding chunks, the panel plays out what it has buffered (PUBLISHED); it then holds the last frame
(INFERRED). A full buffer makes the panel NAK bulk OUT until it drains, so write timeouts must be generous: 2 s per
write, even for a 1 MiB chunk (PUBLISHED, lian-li-linux).

### 6.4 StopPlay (`0x7B`)

No parameters. FIRMWARE: stops the streamed playback. PUBLISHED: sent before every init and before switching from
video to frames, followed by a 150 ms pause. CAPTURED: L-Connect sends it before re-initialising, before an
orientation change and before the switch to monitor mode.

### 6.5 Stream format

| Requirement | Status |
|---|---|
| Annex-B byte stream (start codes), no MP4 container | PUBLISHED; CAPTURED; OBSERVED |
| **480 × 1920** coded size (the SPS must say 480 × 1920) | PUBLISHED (lian-li-linux, hypercolor); INFERRED from the vendor's own clips, which are all 480 × 1920; OBSERVED: every stream played was 480 × 1920. Another size: UNKNOWN (assume rejected) |
| 4:2:0, 8-bit | PUBLISHED; OBSERVED |
| no B-frames | PUBLISHED (every reference encodes with B-frames off); INFERRED from the vendor's clips |
| profile | High works (OBSERVED 2026-11-02; the vendor's clips are High at levels 3.1 and 3.2, HOST, from their headers); Baseline is used by lian-li-linux (PUBLISHED) |
| SPS and PPS before every IDR frame | recommended (PUBLISHED: lian-li-linux and turing-smart-screen-python repeat them); OBSERVED: streams with them play and loop cleanly. Whether a single leading SPS/PPS is enough is UNKNOWN |
| access unit delimiters | optional: lian-li-linux emits them (PUBLISHED); streams without them play (OBSERVED 2026-11-02) |
| rate | 30 fps OBSERVED; the vendor's clips run at 30 and 60 fps (HOST, from their headers) |
| bit rate | about 5 to 10 Mbit/s OBSERVED 2026-11-02 (High profile, 2 s GOP); the decoder's limit is UNKNOWN |

Landscape video is rotated a quarter turn and fitted into 480 × 1920 on the host before encoding (OBSERVED
2026-11-02), as for frames (5.3).

### 6.6 Obfuscated vendor clips

lian-li-linux documents a vendor container that starts with ASCII `lgbl`: XOR with the repeating 8-byte key `lgbm9lgb`
turns it back into Annex-B (the first four bytes become `00 00 00 01`) (PUBLISHED). lcd207 carries the same key string
and methods named `EncVideo` / `DecVideo` (METADATA). The panel itself is fed plain Annex-B (INFERRED); decode on the
host.

## 7. Storage and local playback (FIRMWARE)

The panel has a media tree and an optional SD card. These commands exist in the firmware; L-Connect was not seen using
any of them, and the bench sent only QueryStorage. Names are relative to the firmware's media directory unless they
start with `/` (INFERRED from the handler; an absolute name reaches any file the application can open, including its
own configuration).

**Read-only:**

| Opcode | Name | Parameters | Reply |
|---|---|---|---|
| `0x64` | QueryStorage | none | `[8..11]` total, `[12..15]` used, `[16..19]` free, in KiB, **little-endian**; measured on the SD card mount only. OBSERVED 2026-10-30: all zero on this unit (no SD card fitted) |
| `0x62` | GetFileSize | `[8..11]` name length (BE), `[12..]` name | the file size |
| `0x63` | QueryDir | first call: `[8..11]` name length (BE), `[12..]` directory name | **paged**: each call returns the next 512-byte slice of a 10 240-byte text listing (lines `dir:<name>`, `file:<name>`, `nodir`, `..`), with **no echo or status bytes**; the slice offset wraps to the start after the end |
| `0x70` | GetLocalH264Playing | none | `[8]` = 0 / 1 |
| `0x72` | GetImgPlaying | none | `[8]` = 0 / 1 |

Because the QueryDir pages carry no header, a driver that sends `0x63` must not treat its reply as a normal reply,
and must keep reading pages until it has the listing or it will desynchronise (INFERRED).

**Local playback** (changes what is shown, writes nothing):

| Opcode | Name | Parameters |
|---|---|---|
| `0x6E` | PlayLocalH264 | `[8..11]` name length (BE), `[12..]` name of an H.264 file on the panel |
| `0x6F` | StopLocalH264 | none |
| `0x71` | ShowLocalJpg | `[8..11]` name length (BE), `[12..]` name of a JPEG on the panel |

**Writes and deletion (do not send; listed so that nobody sends them by accident):**

| Opcode | Name | What it does |
|---|---|---|
| `0x26` | CreateFile | opens a named file for writing, truncating it; `[8..11]` name length (BE), `[16..]` name; reply `[8]` = `0xC8` when it opened |
| `0x27` | WriteFileChunk | writes the payload to the file `0x26` opened; `[8..11]` bytes to read from USB, `[12..15]` bytes to write (both BE), `[16]` last-chunk flag |
| `0x28` | UpdateFireWare (a one-shot file write) | `[8..11]` name length, `[12..15]` file length (both BE), `[16..]` name; the data follows. lian-li-linux uses it to upload start-up images to other panels and reports that on this panel such uploads had no effect (PUBLISHED); the firmware's start-up files have other names (10.4) |
| `0x2A` | DelFile | `[8..11]` name length (BE), `[12..]` name; deletes it. (The layout some sources give, with the name at 16, is wrong for this firmware) |

Each write goes to the panel's flash. A wrong name can overwrite the panel's configuration or its start-up files, and
an interrupted transfer leaves a partial file (INFERRED). lian-li-linux refuses every further command after an
interrupted storage write until the device is reconnected (PUBLISHED), which is a sensible guard for anyone who does
implement them.

## 8. Sequences

### 8.1 L-Connect's re-initialisation (CAPTURED, 05:19:14.867, after the return from monitor mode)

| Order | Command |
|---|---|
| 1 | GetVer `0x0A` → `lianli88_0001_0023` |
| 2 | Brightness `0x0E`, 50 |
| 3 | StopPlay `0x7B` |
| 4 | SetClock `0x33`, mode 2, local time |
| 5 | StopClock `0x34` |
| 6 | FrameRate `0x0F`, 7 |
| 7 | the H.264 background: StartPlay `0x79` chunks with QueryBlock `0x7A` flow control |

All replies carried `C8`.

### 8.2 Init used by the open-source drivers (PUBLISHED; OBSERVED with the independent driver)

1. Drain the IN pipe.
2. StopPlay, read its reply, wait 150 ms (stops whatever another host left playing).
3. GetVer; keep the string.
4. FrameRate (lian-li-linux: 120).
5. Brightness at the wanted level (hypercolor does, lian-li-linux does not; send it, or a panel left at 0 stays dark).
6. SetClock mode 2 with local time.
7. StopClock.
8. PushPng of a transparent 480 × 1920 PNG, then PushJpg of a black 480 × 1920 JPEG (blanks both layers).

OBSERVED: the independent driver sent only GetVer when it opened the panel (nothing changes on the screen), and ran
steps 2 and 4 to 8 before its first frame, with a pause of about 250 ms after StopClock (4.7). lian-li-linux and the
independent driver use 2 s write and read timeouts at init.

### 8.3 Orientation change (CAPTURED)

- **180° flip** (the "180" button): one Rotate, `1` or `3`, with the stream left running (05:20:02.8, 05:20:05.8).
- **Landscape → portrait** (05:20:14.778 to 05:20:15.1): StopPlay, Rotate `2`, StopPlay, SetClock mode 2, StopClock,
  then a new stream at FrameRate 7 rendered for portrait.
- **Portrait → landscape** (05:21:04.529 to 05:21:04.9): the same with Rotate `3`.

### 8.4 Show a picture

1. Init done (8.2). If H.264 is playing, StopPlay first.
2. Encode a baseline 480 × 1920 JPEG of at most 512 000 bytes.
3. One write: PushJpg header + JPEG.
4. Read the reply; if `[8]` > 3, poll QueryBlock until ≤ 2 (5.4).

### 8.5 Stream a video

1. Init done; StopPlay; wait 150 ms; FrameRate = the clip's rate.
2. StartPlay chunks of 202 752 bytes with the flow control of 6.3.
3. To stop: StopPlay (and 150 ms before the next command).

Commands other than chunks during a stream: L-Connect sent SetClock and Rotate while its stream ran (CAPTURED). On a
sibling Lian Li product, lian-li-linux saw control writes during heavy H.264 ingest hang the controller, and it sends
other commands only between chunks when the buffer level is 1 or less (PUBLISHED). Whether that hazard exists on this
panel is UNKNOWN; the same rule is cheap.

### 8.6 Leaving

- To leave the last picture up: StopPlay if streaming, then close the handle (PUBLISHED, hypercolor sends nothing).
- To blank: StopPlay, then Brightness 0 (PUBLISHED, lian-li-linux's default at shutdown). Brightness 0 is not saved
  (4.4), so the panel lights up again at its saved level after a power cycle, but a running host session that sets 0
  leaves it dark until someone sends a brightness.
- There is no command that returns the panel to its built-in start-up animation; a reboot or power cycle does
  (FIRMWARE, section 10.4).

## 9. Windows monitor mode and back

The panel can re-enumerate as a USB display that Lian Li's Windows display driver presents as a 1920 × 480 monitor.
In monitor mode `1CBE:A088` is gone and none of the commands above are available.

### 9.1 LCD mode → monitor mode

CAPTURED (L-Connect's "Desktop mode" button, all four commands answered `C8`):

| Time | Command |
|---|---|
| 05:18:27.066 | StopPlay `0x7B` |
| 05:18:27.079 | SwitchToDesktop `0x96` (no parameters) |
| 05:18:27.090 | Reboot `0x0B` |
| 05:18:27.125 | QueryBlock `0x7A` |

The hub reported the port change at 05:18:29.68; `1A86:AD21` enumerated at **05:18:31.269, 4.2 s after Reboot**.
PUBLISHED (lian-li-linux): the same pair, `0x96` then `0x0B`, after stopping playback. OBSERVED 2026-10-30 06:23:36:
the same pair sent by the independent driver; within 8 s Windows showed the 1920 × 480 display with Lian Li's driver
("LIANLI USB Secondary Display-60hz") bound. The same again on 2026-10-31 17:45:40 (monitor device up 7 s later).

Rev1 units come back as `1A86:ACE1` instead (PUBLISHED, lian-li-linux). Whether Lian Li's Windows driver supports
them is UNKNOWN, and the two revisions cannot be told apart in LCD mode (same `1CBE:A088`, INFERRED). Do not switch a
unit to monitor mode unless you can switch it back.

### 9.2 The monitor-mode device (CAPTURED)

| Item | Value |
|---|---|
| Configuration | `09 02 78 00 02 01 04 80 FA`: two interfaces |
| Interface 0 | class `FF/FF`, with a vendor descriptor of type `0x5F` embedded; bulk OUT `0x01`, bulk IN `0x81`, 512 bytes |
| Interface 1 | **HID**, interrupt OUT `0x02` and interrupt IN `0x82`, 512 bytes, interval 4 |
| HID report descriptor | `06 00 FF 09 00 A1 01 15 00 25 FF 19 00 29 FF 75 08 96 00 02 91 02 19 00 29 FF 75 08 96 00 02 81 02 C0`: vendor page, one 512-byte output report and one 512-byte input report, **no report IDs** |
| Strings | product "USB Display", interface 1 "HID Interface" |

On Windows, the HID output report is therefore written as a 513-byte buffer: report ID `00` + 512 bytes (INFERRED from
the descriptor; OBSERVED: such writes work, 9.3). The wire carries the 512 bytes without the ID byte (CAPTURED).

**Identity query (CAPTURED, 0.6 s after enumeration, sent by the vendor software or its display driver):** output report
`AA 55 35 00` followed by zeros; the input report answers with the 16 bytes GetChipId returns in LCD mode (`<serial>`
then the same 8 unknown bytes, then zeros). Not in any open-source project.

**Display start-up (CAPTURED; lian-li-linux implements the same TURZX protocol for Linux, PUBLISHED):** control IN
"ready" poll (`bmRequestType C1`, `bRequest 01`) answered `18` (bit `0x10` = ready); the vendor descriptor read as a
standard GET_DESCRIPTOR for type `0x5F` on the interface (`81 06 wValue 0x5F00`, 56 bytes); the EDID (`C1 02`, 128
bytes); then a bulk configuration packet of `AF 20 <sub-op> <value>` records and an H.264 stream in frames with opcode
`0x6D`. The values are in section 13.3. At teardown the driver sends the power-off record `AF 20 1F 02`.

The vendor descriptor and EDID say: minimum 240 × 240, maximum 2047 × 2047, maximum transfer 131 040 bytes, MJPEG and
H.264, modes 1920 × 480 at 60 and 30 Hz (CAPTURED; parsed with lian-li-linux's documented TLV format, PUBLISHED; the
same bytes are a committed test fixture there). The configuration record `05 64` (value 100) is not in lian-li-linux;
INFERRED to be the backlight. Brightness, rotation and DDC/CI in monitor mode are UNKNOWN.

### 9.3 Monitor mode → LCD mode

CAPTURED (L-Connect's "LCD mode" button, 05:19:07.916): **one 512-byte output report on interrupt OUT `0x02`:
`35 66 33 37 35 39 64 66` followed by 504 zero bytes** (ASCII `5f3759df`). The device dropped off the hub at
05:19:10.61, and `1CBE:A088` enumerated at **05:19:12.161, 4.2 s after the report**.

PUBLISHED: lian-li-linux sends the same 8 ASCII bytes with a HID write (and a leading `00` only for another product);
lianli-python-lcd-overlay writes them, padded to 512 bytes, straight to endpoint `0x02`. On Windows, send report ID
`00` + the 8 bytes, zero padded to the 513-byte output report length (OBSERVED 2026-10-30 06:24:06 and 2026-10-31:
`1CBE:A088` back within 10 s; the panel showed its start-up logo in between). lian-li-linux stops its monitor-mode
stream (power-off record) and waits 300 ms before the switch (PUBLISHED); L-Connect's display driver sent its
power-off in the same millisecond as the device left (CAPTURED).

The panel came back with the orientation it had before (CAPTURED: rotation 3 kept). L-Connect then re-initialised it
(8.1).

### 9.4 What the switch does inside the panel (FIRMWARE)

SwitchToDesktop `0x96` only creates an empty flag file (`/usr/data/usb_display`). At the next boot the start-up
scripts see the flag and start the display gadget instead of the serial one. Reboot `0x0B` is what makes it happen.
Hence Reboot alone (no flag) returns in LCD mode, and `0x96` alone changes nothing until the next reboot or power
cycle (INFERRED: the flag would then still take effect, so do not send `0x96` unless Reboot follows). The return
path is in the monitor-mode software, which is not part of the application read here; INFERRED: it removes the flag
and reboots when it receives the magic report.

## 10. Standalone behaviour

### 10.1 Host attached but silent

FIRMWARE: there is **no timeout on host traffic**. While the USB link is up, the panel keeps the last frame. When the
link has just come up (after a boot or a reconnect) it plays the saved "mode" preset until the host sends frames:
mode 1 = the local image player (images in the panel's image folders), mode 2 = a local H.264 file (its name is saved
too; default `offline_video.h264` in the media tree), otherwise just the backlight at the active level. PUBLISHED
(hypercolor): the panel keeps the last frame after the host stops. The preset was never tried (only SetConfig `0x7D`
sets it, 4.9).

### 10.2 Host gone: offline clock or dim

FIRMWARE: when the device controller's state leaves `configured` and stays out for 10 polls (about 10 s), the panel:

- with the offline clock **on** (SetClock mode 1): draws a clock itself, time `hh:mm:ss` and a date (three numbers
  separated by `/`, order UNKNOWN), in a TrueType font over a logo image, laid out for landscape or portrait by the
  orientation hint of 4.3;
- with it **off** (mode 0): dims to an "inactive" backlight level (value UNKNOWN) and stops any local video.

"Link gone" includes the PC sleeping with USB power on (OBSERVED, below) and, INFERRED, the PC shut down with standby
power still on the USB header, or a host that deconfigures the device. A host that merely stops sending does not count
(10.1).

OBSERVED 2026-10-30 06:38 to 06:41: offline clock set on with SetClock mode 1, PC put into S3 sleep (USB power stays
on in S3 on the test PC): the panel drew its clock during the sleep, and after the resume it showed the host's
picture again 9 s later, once the driver had reopened the handle. The first GetVer and the first frame after the
resume went unanswered (4.7, 2.5).

When the link comes back, the first command stops the clock (4.7).

### 10.3 Saved settings (FIRMWARE)

The firmware keeps a small configuration file (`/usr/data/app.cfg`) and writes it with `sync` after every change:

| Saved | Set by | Applied |
|---|---|---|
| brightness (only values above 0) | Brightness `0x0E`, SetConfig `0x7D` | at boot |
| "mode" preset | SetConfig `0x7D` | on link-up (10.1) |
| orientation value and rotator angle | Rotate `0x0D`, SetConfig `0x7D` | at boot |
| offline clock on/off | SetClock mode 0 / 1, SetConfig `0x7D` | when the link drops (10.2) |
| test mode | factory | reset to 0 by the first command |

**Not saved:** frame rate (boot default 30), brightness 0, anything streamed. Whether the clock time survives a power
cut is UNKNOWN; every host sets it at every init (CAPTURED, PUBLISHED).

Each of these writes flash. Sending Brightness or Rotate many times a second wears it (INFERRED); send them on change.

### 10.4 Boot (FIRMWARE)

At start the application reads the chip id and its configuration, sets up the framebuffers (frame rate 30), applies
the saved brightness, measures the SD card, starts its threads, plays the start-up animation (`boot.h264` in the media
tree, else the logo `logo.jpg`), then the saved preset, then starts the clock and opens the serial gadget. OBSERVED
2026-10-30: the start-up logo showed while the panel returned from monitor mode; after a full power
cycle with the PSU switched off (2026-10-31) the panel came back and accepted the driver's re-initialisation with no
help.

The start-up files have different names from the `boot.jpg` the open-source projects upload to other Lian Li panels
(PUBLISHED, lian-li-linux), which fits their report that such uploads had no effect here (INFERRED). Replacing the
start-up files would be a file write of section 7; this reference does not describe it as a feature.

### 10.5 Power events with a host

OBSERVED: after an S3
resume the panel is still enumerated (USB power stayed on), the old handle may fail on its first write, and a close
and reopen followed by the normal init restores the picture. After the panel re-enumerates (monitor-mode round trip,
reboot, power cycle) a fresh open and init is needed.

## 11. Error recovery

PUBLISHED (lian-li-linux), OBSERVED with the independent driver:

- Recover by **closing and reopening** the handle (wait about 100 ms in between), clearing any halt on the two pipes,
  and forgetting the negotiated chunk size. Never reset the USB port or the hub: the LED ring shares the hub inside the
  enclosure.
- A failed or timed-out write may already have delivered the header and part of its payload, and the panel may still
  be waiting for the rest. After a write error, reopen first, then resend the **whole** packet (header and payload)
  or give it up; never put a new header on the same handle (PUBLISHED; INFERRED from the firmware reading the stated
  payload length after each header).
- Resend at most once. lian-li-linux resends a failed H.264 chunk at most once per 60 s.
- Write timeout 2 s for everything, including 512 KB frames and 1 MiB chunks (PUBLISHED).
- If the panel is not enumerated at `1CBE:A088`, check for `1A86:AD21` / `1A86:ACE1`: it may be in monitor mode
  (section 9).

## 12. Commands that must not be sent

| Opcode | Name | Why |
|---|---|---|
| firmware update | L-Connect's "StartFirmwareUpdate" (METADATA request type 4) and whatever it sends | re-flashes the panel; the sequence was deliberately not captured; a failure can brick it. Leave updates to the vendor tool |
| `0x2B`, `0x2C` | OTA, RebootApp (METADATA) | update-related names in the vendor enum; not handled by this application (FIRMWARE), so their effect would come from elsewhere in the update path (UNKNOWN). Never send |
| `0x26`, `0x27`, `0x28` | file writes | write any file the application can open, including its start-up files and the files under `/usr/data` (the saved configuration, the monitor-mode flag); an absolute name is not confined to the media tree (section 7) |
| `0x2A` | DelFile | deletes a named file; could remove start-up files or the configuration |
| `0x0C` | Halt | shuts the panel's Linux down; it stays off until power is cut |
| `0xFD` | TestH264 / FactoryH264Test | factory test: starts a test decoder on a fixed test file |
| `0x97`, `0xFE` | Test, test-flag reset | factory |
| `0x7D` | SetConfig | writes six saved fields at once, two of unknown meaning (4.9) |
| `0x0B` (alone), `0x96` | Reboot, SwitchToDesktop | not dangerous, but the panel leaves the bus; send only as the deliberate mode switch of section 9 |
| `0x60`, `0x61` | GetTemperature, SetPumpSpeed | AIO pump-head commands of other products; not handled here, no reason to send |
| TURZX storage opcodes | `0x28` "delete" and `0x62` "play" of turing-smart-screen-python | mean file write and GetFileSize here (3.1) |

Also do not probe: every opcode is answered with `C8` (2.5), so probing learns nothing and some opcodes act at once.

## 13. Test vectors

### 13.1 Header encryption

All values computed from the layout in section 2 (key = IV = `slv3tuzx`, PKCS#7, trailer `A1 1A`, timestamp LE, sizes
BE) with three independent DES implementations. The GetVer and PushJpg vectors equal the vectors committed in
hypercolor's tests (PUBLISHED). Plaintext is shown from byte 0; wire bytes 0..15 and 496..511.

```
GetVer ts=1
  plaintext[0..16]: 0a 00 1a 6d 01 00 00 00 00 00 00 00 00 00 00 00
  wire[0..16]:      f1 32 a5 d4 e3 cf f8 57 48 b8 2a aa ca c6 8f 8a
  wire[16..32]:     e7 6b d5 35 d3 e9 53 8a 13 0a 7b 11 21 9f 36 5c
  wire[488..496]:   9d 69 87 a8 6a 3d 99 54
  wire[496..512]:   b1 3e c4 2b b4 83 67 04 00 00 00 00 00 00 a1 1a

PushJpg ts=42 size=123456 (00 01 e2 40 BE)
  plaintext[0..16]: 65 00 1a 6d 2a 00 00 00 00 01 e2 40 00 00 00 00
  wire[0..16]:      6e df 63 cc 8e 6c 3f e4 3f 32 42 8b 39 cc e2 2c
  wire[488..496]:   a5 6d ea f6 02 d3 4c 4d
  wire[496..512]:   27 38 27 32 b8 3e f6 43 00 00 00 00 00 00 a1 1a

Brightness ts=2 value=50
  plaintext[0..16]: 0e 00 1a 6d 02 00 00 00 32 00 00 00 00 00 00 00
  wire[0..16]:      ac 72 10 2b c9 6c 30 6b 29 2c 10 5b dc 0e 50 ab
  wire[496..512]:   7d 2e a3 df 44 6b 3b 2d 00 00 00 00 00 00 a1 1a

FrameRate ts=3 fps=120 (0x78)
  plaintext[0..16]: 0f 00 1a 6d 03 00 00 00 78 00 00 00 00 00 00 00
  wire[0..16]:      97 6a dd 52 84 49 2b 41 d5 d7 ef 0d e9 4f ef fd
  wire[496..512]:   89 85 69 50 aa eb 09 04 00 00 00 00 00 00 a1 1a

SetClock ts=4 2026-10-08 19:05:42 mode=2
  plaintext[0..16]: 33 00 1a 6d 04 00 00 00 07 ea 0a 08 13 05 2a 02
  wire[0..16]:      24 1a aa 0b 82 6b c4 78 3b c2 08 fd 21 e4 88 8e
  wire[496..512]:   67 ae 0c fa 32 a8 8e 37 00 00 00 00 00 00 a1 1a

StopClock ts=5
  plaintext[0..16]: 34 00 1a 6d 05 00 00 00 00 00 00 00 00 00 00 00
  wire[0..16]:      66 58 61 9e 1a cc 5e ef ed a7 07 ec 5f 7f c5 c3
  wire[496..512]:   f6 67 8f 47 d8 cf 02 49 00 00 00 00 00 00 a1 1a

StopPlay ts=7
  plaintext[0..16]: 7b 00 1a 6d 07 00 00 00 00 00 00 00 00 00 00 00
  wire[0..16]:      6c fd be 4d e3 f8 9f ca 4a 11 45 4d c9 80 f2 fb
  wire[496..512]:   db e0 d4 27 48 a8 a2 37 00 00 00 00 00 00 a1 1a

QueryBlock ts=8
  plaintext[0..16]: 7a 00 1a 6d 08 00 00 00 00 00 00 00 00 00 00 00
  wire[0..16]:      12 95 17 64 92 fa c2 97 da 60 33 c6 cd 85 aa c7
  wire[496..512]:   68 4d 1f fa 93 5a 7c 08 00 00 00 00 00 00 a1 1a

SwitchToDesktop ts=10
  plaintext[0..8]:  96 00 1a 6d 0a 00 00 00
  wire[0..16]:      c5 f5 89 fd 8c d4 c5 c2 c1 be db e5 9a 26 3f 0e
  wire[496..512]:   93 13 24 cc 6c 90 16 bf 00 00 00 00 00 00 a1 1a

Reboot ts=11
  plaintext[0..8]:  0b 00 1a 6d 0b 00 00 00
  wire[0..16]:      cb 09 a0 09 58 8b c4 ce 38 4b 8f f4 1f 81 5e 65
  wire[496..512]:   dc 9d a5 37 a9 a5 97 9c 00 00 00 00 00 00 a1 1a

GetH264Block ts=13
  plaintext[0..8]:  11 00 1a 6d 0d 00 00 00
  wire[0..16]:      b6 df c5 44 76 0d f0 5e 91 aa eb 4c 04 71 93 75
  wire[496..512]:   eb c6 89 cf 60 1e 52 ac 00 00 00 00 00 00 a1 1a

StartPlay ts=100 len=202752 (00 03 18 00 BE) last=0 count=0 tick=0
  plaintext[0..16]: 79 00 1a 6d 64 00 00 00 00 03 18 00 00 00 00 00
  wire[0..16]:      31 82 fa 5d a4 c0 6f b8 d1 1f 81 9b 6a 3b 85 ff
  wire[496..512]:   d4 89 00 6f 90 5b 55 0a 00 00 00 00 00 00 a1 1a
```

With turing-smart-screen-python's zero padding instead of PKCS#7, GetVer ts=1 differs only in `wire[496..504]` =
`25 ed 2b 0b df 5b e3 81` (for comparison; not what the vendor sends).

hypercolor's committed clock test: 2026-10-08 19:05:42 → SetClock parameters `07 EA 0A 08 13 05 2A 02` (PUBLISHED), as
in the vector above.

### 13.2 Replies (FIRMWARE layout; values CAPTURED or OBSERVED)

```
GetVer reply     0A C8 tt tt tt tt 00 00  6C 69 61 6E 6C 69 38 38 5F 30 30 30 31 5F 30 30 32 33 00 ...
                                          "lianli88_0001_0023"
QueryStorage     64 C8 tt tt tt tt 00 00  00 00 00 00  00 00 00 00  00 00 00 00 ...   (no SD card)
```

`tt tt tt tt` = the request's timestamp bytes.

### 13.3 Monitor mode (CAPTURED, this unit)

```
Vendor descriptor (56 bytes):
  38 5F 01 00 36 01 00 04 F0 00 F0 00 02 00 04 FF 07 FF 07 03 00 04 E0 FF 01 00 00 01 04 01 00 00
  40 00 01 04 02 00 00 40 00 02 05 80 07 E0 01 3C 00 02 05 80 07 E0 01 1E

EDID (128 bytes):
  00 FF FF FF FF FF FF 00 52 B2 00 00 00 00 00 00 01 23 01 03 80 01 01 00 08 5E C4 A4 59 4A 98 25
  20 50 54 00 00 00 01 01 01 01 01 01 01 01 01 01 01 01 01 01 01 01 B8 0B 80 40 70 E0 12 10 10 10
  35 00 4D D0 10 00 00 10 0C 17 80 40 70 E0 12 10 10 10 35 00 4D D0 10 00 00 10 00 00 00 10 00 00
  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 10 00 00 00 00 00 00 00 00 00 00 00 00 00 C8

Configuration packet (Windows driver):
  AF 20 00 02  AF 20 01 07  AF 20 02 80  AF 20 03 01  AF 20 04 E0  AF 20 06 00  AF 20 12 01  AF 20 1F 01  AF 20 05 64

Power off:          AF 20 1F 02
Switch back:        35 66 33 37 35 39 64 66 + 504 x 00   (interrupt OUT 0x02)
Identity query:     AA 55 35 00 + 508 x 00
```

EDID manufacturer `52 B2` = "TUR"; two 1920 × 480 detailed timings, the second at half the pixel clock. lian-li-linux
reports this EDID's timings as broken and synthesises its own for Linux (PUBLISHED). In the configuration packet:
`00 02` where lian-li-linux sends `00 01` for "start", width `07 80` = 1920, height `01 E0` = 480, `12 01` selects
H.264 (codec `0x0112`; lian-li-linux uses `11 01` for MJPEG), `1F 01` = power on, `06 00` and `05 64` are not in
lian-li-linux (meaning UNKNOWN; `05 64` INFERRED backlight 100).

## 14. Open questions

| # | Question | How to settle |
|---|---|---|
| 1 | Bus speed of LCD mode (high speed INFERRED) | read the device's speed in a USB descriptor viewer |
| 2 | Is a wrong-geometry JPEG unanswered (hypercolor) or answered (firmware reading)? | send one 1920 × 480 JPEG and watch the reply |
| 3 | Highest sustained JPEG and H.264 rates; whether FrameRate affects JPEG | measure |
| 4 | JPEG 4:2:2 / 4:4:4, progressive; H.264 levels and bit rates beyond those used | try, one at a time |
| 5 | PNG overlay over a playing H.264 stream | push a PNG during a stream |
| 6 | Whether control commands during heavy H.264 ingest are safe (L-Connect sends SetClock and Rotate mid-stream) | send Brightness at buffer level 3 to 4, watch for a hang |
| 7 | Reboot alone returns in LCD mode (FIRMWARE) | not yet seen on the hardware |
| 8 | Meaning of the 8 bytes after the chip id (GetChipId, monitor-mode identity query) | other units |
| 9 | Offline clock: the date order, the inactive backlight level, behaviour with the PC fully off | look at the panel with the PC asleep or shut down |
| 10 | The "mode" preset (local image player, local H.264) and how it interacts with a host | only via SetConfig or storage commands; not recommended |
| 11 | The "secondary screen" setting of L-Connect (METADATA request type 24) | capture its toggle |
| 12 | Monitor-mode brightness, rotation and DDC/CI; the `06 00` and `05 64` configuration records | capture with Lian Li's display driver bound |
| 13 | Rev1 units (`1A86:ACE1`): Windows support, how to tell them apart in LCD mode | a rev1 unit |
| 14 | lianli-python-lcd-overlay's unanswered PushJpg above about 2 KB under libusb | reproduce on Linux |
| 15 | What the monitor-mode side does with the flag file | read the rest of the firmware package, or observe |
