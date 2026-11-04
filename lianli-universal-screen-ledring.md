# Lian Li Universal Screen 8.8": LED ring protocol (USB `0416:8050`)

Part of the [Lian Li Universal Screen 8.8" protocol reference](README.md). Tags (**CAPTURED**, **METADATA**,
**PUBLISHED**, **OBSERVED**, **INFERRED**, **UNKNOWN**) and the capture are explained there. **FIRMWARE** does not
appear in this document: the ring's own firmware was not read. Times are local time of the capturing PC (UTC-7).
"The capture" is the USBPcap session of 2026-10-30, 05:17:33 to 05:29:13, in which L-Connect 3 drove the ring at USB
device address 17. Byte 0 of every transfer is called `[0]`, byte 1 `[1]`, and so on.

Short version (each point is tagged in the sections below): the ring is its own vendor-class USB device (not HID) with one bulk OUT and one bulk IN endpoint
and plain 64-byte packets: no report ID, no encryption, no checksum. Two commands are known and used: `0x11` sets
20 LEDs at an offset, `0x10` reads the firmware version. Every effect the vendor software offers is rendered on the
PC and streamed as `0x11` frames; the ring has no effect, brightness, save or off command that anyone has seen.

## 1. Device

### 1.1 Place in the enclosure

The Universal Screen 8.8" contains three USB functions behind an internal WCH hub: the LCD panel (`1CBE:A088`), this
LED ring (`0416:8050`) and the hub itself (**PUBLISHED**, hypercolor's driver specification for this product). The
hub on the captured unit is `1A86:8091`; hypercolor's reporter had `1A86:8095`, so the hub PID varies between units
(**PUBLISHED**, hypercolor). The ring has its own MCU and its own USB identity, and it does **not** use the LCD's
DES-encrypted command header (**PUBLISHED**, lian-li-linux, OpenRGB and lianli-python-lcd-overlay all send plain
bytes; **CAPTURED**: every ring transfer in the capture is plaintext).

### 1.2 Descriptors

**CAPTURED (2026-10-30, 05:17:33.362)**, from the descriptors USBPcap injected at the start of the capture. The same
device descriptor and pipe list were read by third-party code on 2026-10-27 (**OBSERVED**).

| Item | Value |
|---|---|
| Device descriptor | `12 01 01 02 FF FF FF 40 16 04 50 80 00 00 01 02 00 01`: bcdUSB 2.01, device class `FF/FF/FF` (vendor), 64-byte EP0, VID `0416`, PID `8050`, bcdDevice `0000`, manufacturer string index 1, product string index 2, **no serial-number string** (index 0), one configuration |
| Configuration | `09 02 20 00 01 01 00 A0 23`: one interface, configuration value 1, bus powered with remote wake-up (`A0`), bMaxPower 35 = 70 mA |
| Interface 0 | `09 04 00 00 02 FF 00 00 00`: vendor class `FF/00/00`, two endpoints, no string |
| Endpoint `0x81` | `07 05 81 02 40 00 00`: bulk IN, 64 bytes |
| Endpoint `0x01` | `07 05 01 02 40 00 00`: bulk OUT, 64 bytes |

Full configuration descriptor as captured:

```
09 02 20 00 01 01 00 A0 23
09 04 00 00 02 FF 00 00 00   07 05 81 02 40 00 00   07 05 01 02 40 00 00
```

- **Product name.** Windows lists the ring with the bus-reported description `8.8" LED-1.0` (HOST). INFERRED: this is the
  product string (index 2); the string descriptor itself was not captured.
- **Bus speed.** A bulk endpoint with a 64-byte maximum packet is legal only at full speed (USB 2.0 requires 512
  bytes for high-speed bulk), so the ring runs at full speed (INFERRED). The LCD on the same hub is a different
  device and uses 512-byte packets.
- **The 70 mA in the configuration** is what the MCU declares; how the 60 LEDs are powered is UNKNOWN.
- **No serial number.** Two screens on one PC cannot be told apart by a USB serial (CAPTURED: index 0). Identify a
  ring by its place in the USB tree and pair it with the LCD that sits behind the same hub (INFERRED; lian-li-linux
  identifies the ring by bus and port path, **PUBLISHED**). The LCD has a serial; the ring does not.
- **MCU.** UNKNOWN. OpenRGB names VID `0x0416` after Nuvoton, hypercolor after Winbond (**PUBLISHED**); the silicon
  was not identified. lianli-python-lcd-overlay calls it a "WCH chip", but WCH parts use VID `0x1A86` (the hub's), so
  that label is probably wrong (INFERRED).
- **Firmware.** The captured unit reports `lianliLed88_1_3` (section 5).

### 1.3 The vendor's view (METADATA)

From the .NET metadata of `lianli.lcd207.dll` 1.4.0.125 and `L-Connect.Core.dll` (L-Connect 3):

- The ring's transport class is `lcd207.WinUsbLed`, with constants `LED_88inchVid = 1046` (`0x0416`) and
  `LED_88inchPid = 32848` (`0x8050`). It derives from the LCD transport base class `lcd207.WinUsb` and declares
  methods `GetAllLedDevice`, `ReInitDev`, `GetLedVer`, `Test` and `UpdateLed`. A controller class `lcd207.LEDController`
  declares `GetLedVer`, `SetEffect`, `StopEffect`, `CheckLedUpdate`, `UpdateLed` and a `NewFWPath` property.
- The lighting settings type `UniversalScreen8p8InchLightingSetting` has the fields `Colors`, `Brightness`, `Speed`
  and `IsReverseDirection`. The mode enum `UniversalScreen8p8InchLightingMode` has 19 values, `Rainbow = 0` to
  `RainbowWave = 18` (section 7.4). It has no "Off" value.
- The version string is validated against the user string `^lianliLed88_(\d{1,3})_(\d{1,3})$` (section 5).
- A nested enum `lcd207.WinUsbLed+USB_CMD` lists command values, among them firmware erase, upload, verify and reboot
  (section 6). Its names for `0x10` and `0x11` (`Usb_SendRf`, `USB_GetMac`) look borrowed from Lian Li's RF dongles
  and do not describe what those values do on the ring (INFERRED). Only the numbers match the protocol.

Other Lian Li devices share VID `0x0416`, so matching the VID alone is wrong (**PUBLISHED**, lian-li-linux's device
list, plus METADATA for `0x8052`):

| PID | Device |
|---|---|
| `0x8040`, `0x8041` | wireless dongles |
| `0x7372` | TL fan hub |
| `0x7371`, `0x7373` | GA II |
| `0x7391`, `0x7395` | Galahad II LCD / Vision |
| `0x7398`, `0x7399`, `0x739A` | HydroShift LCD AIO |
| `0x8051` | HydroShift II OLED, LED MCU (also uses `0x10` / `0x11`, but with 8-byte commands and 15 LEDs per packet) |
| `0x8052` | a second HydroShift II LED PID (METADATA, `lcd207.WinUsbHS2`, constant `LED1_HS2Pid = 32850`) |

## 2. Transport

| Item | Value | Source |
|---|---|---|
| Device type | vendor class, single interface 0; **not HID**, so there are no report IDs | CAPTURED (descriptors); the HID-style names some projects use for it are wrong |
| OUT endpoint | `0x01`, bulk | CAPTURED; PUBLISHED by lian-li-linux, OpenRGB and lianli-python-lcd-overlay |
| IN endpoint | `0x81`, bulk; carries only the `0x10` reply | CAPTURED |
| Transfer size | exactly 64 bytes, every transfer | CAPTURED: all 45 286 OUT transfers of the capture are 64 bytes |
| Framing | none: `[0]` is the opcode; no header, no length field, no checksum, no encryption | CAPTURED |
| Endianness | not applicable: the known commands have only single-byte fields | CAPTURED |
| Configuration | configuration 1, alternate setting 0. Windows selects it; do not send SET_INTERFACE | CAPTURED (descriptors); INFERRED for SET_INTERFACE (no known host sends it, lian-li-linux deliberately avoids it, **PUBLISHED**) |

**Windows.** The ring binds to the inbox WinUSB driver by itself through Microsoft OS descriptors; no INF is
involved (HOST: the compatible id `USB\MS_COMP_WINUSB` Windows lists for the device; INFERRED from bcdUSB 2.01,
which is what a device that offers MS OS 2.0 descriptors in its BOS reports). The device-interface GUID Windows
registers for it, `{1D4B2365-4749-48EA-B38A-7C6FDDDD7E26}`, comes from those descriptors (HOST; INFERRED that it is the same on every unit, one unit seen;
read it from the device's properties rather than hard-coding it). Open the interface path with `CreateFile` and
`WinUsb_Initialize`, write with `WinUsb_WritePipe(0x01)`, read with `WinUsb_ReadPipe(0x81)`, and set
`PIPE_TRANSFER_TIMEOUT` per pipe so a hung pipe cannot block forever (INFERRED, ordinary WinUSB practice). On Linux,
libusb on interface 0 works (**PUBLISHED**, lian-li-linux and lianli-python-lcd-overlay).

**Exclusive access.** WinUSB lets one process hold the device. In L-Connect 3 the holder is its background
service, not its window: closing the L-Connect window changed nothing on the wire (CAPTURED, 2026-10-30). On the
bench every L-Connect process (the service and the window process it starts at logon) had to be stopped before
third-party code took the ring over (OBSERVED, 2026-10-27 to 2026-10-31); that a second open fails while another
process holds the handle is ordinary WinUSB behaviour (INFERRED, not tried).
OpenRGB, from its December 2025 Universal Screen controller on, also opens the ring (**PUBLISHED**, OpenRGB), so it
competes for the same handle.

**Timing on the wire (CAPTURED).** Each 64-byte transfer completed 0.1 ms (median) after submission, at most
1.2 ms; all 45 286 completed with status 0. The three packets of one frame left within 0.27 ms of each other
(median), at most 12.6 ms.

## 3. Command set

| `[0]` | Name | Direction | Request | Reply | Status |
|---|---|---|---|---|---|
| `0x11` | SetLeds | OUT only | `11`, first LED, `00`, `00`, 20 × RGB | none | CAPTURED (45 285 packets = 15 095 frames); PUBLISHED by lian-li-linux, OpenRGB and lianli-python-lcd-overlay; OBSERVED 2026-10-27 |
| `0x10` | GetVersion | OUT, then IN | `10` + 63 × `00` | `10 <major> <minor>` + 61 × `00` | CAPTURED (once, 05:19:14.878); PUBLISHED by lianli-python-lcd-overlay; OBSERVED 2026-10-27 |
| anything else | | | | | **do not send** (section 6) |

In 11.6 minutes of L-Connect driving the ring through effect, speed and brightness changes, a 0 % brightness
setting, a screen switch to monitor mode and back, and the closing of its window, these two opcodes were the only
ones on the wire (CAPTURED).

## 4. `0x11` SetLeds

### 4.1 Layout

| Offset | Size | Field | Value |
|---|---|---|---|
| 0 | 1 | opcode | `0x11` |
| 1 | 1 | index of the first LED in this packet | `0x00`, `0x14` or `0x28` (0, 20, 40) |
| 2 | 1 | reserved | `0x00` |
| 3 | 1 | reserved | `0x00` |
| 4 + 3*i | 1 | red of LED `first + i` | 0-255 |
| 5 + 3*i | 1 | green | 0-255 |
| 6 + 3*i | 1 | blue | 0-255 |

`i` runs 0 to 19, so bytes 4 to 63 carry 20 LEDs. A frame is three packets, offsets 0, 20 and 40, 60 LEDs in all.

- **Source.** **PUBLISHED** by lian-li-linux (`crates/lianli-devices/src/winusb/led.rs`), OpenRGB (its Lian Li
  Universal Screen controller) and lianli-python-lcd-overlay (`Linx/linx.py`), which agree byte for byte.
  Re-checked against the capture: every frame packet matches this layout (CAPTURED).
- **Offsets (CAPTURED).** L-Connect sent the offsets 0, 20, 40, always all three and always in that order (15 095
  complete triplets, no exception). Other offsets, and an offset above 40, are UNKNOWN.
- **Bytes 2 and 3 (CAPTURED).** `00 00` in all 45 285 packets. Their meaning on this firmware is UNKNOWN. In a
  related Lian Li controller family (wired receivers `43A8:0101` to `0107`) the same packet uses `[2]` = number of
  LEDs in the packet and `[3]` = `0x02` on the last packet of a frame (**PUBLISHED**, lian-li-linux's wired-receiver
  driver); nothing shows that the ring honours either, so send `00 00`.
- **Colour order R, G, B** (PUBLISHED by all three projects; OBSERVED 2026-10-27: whole-ring red, green, blue, white
  and black frames from third-party code showed the intended colours, with no flicker).
- **No brightness byte.** The ring has no brightness field or command. L-Connect's brightness slider only scales the
  colour values in the frames, and 0 % produced all-zero frames (CAPTURED: 136 all-black frames from 05:25:21.537).
  Dimming is the host's job.
- **No reply (CAPTURED, OBSERVED).** After the `0x10` reply, L-Connect kept a read pending on IN `0x81` from
  05:19:15.080 to the end of the capture (about 10 minutes and some 12 000 frames); nothing arrived. Third-party code
  then read IN `0x81` deliberately on 2026-11-03: twice with a 500 ms timeout while frames streamed every 62 ms, and
  twice with 50 ms plus once with 500 ms right after a single frame; nothing arrived. `0x11` produces no IN data
  (OBSERVED). The only IN data the ring ever sent is the `0x10` reply. Never wait for a reply after a frame.
- **Each packet takes effect on its own (OBSERVED 2026-11-03).** Third-party code sent, from a uniform frame, single
  packets that changed only one chunk: offset 0 alone, offset 20 alone and offset 40 alone (and pairs 0+20 and 20+40).
  Each one moved the lit LED at once; no end-of-frame signal exists with bytes 2-3 at `00 00`. So a partial update of
  20 LEDs is one packet, and a frame whose three packets are spread out in time shows old and new chunks mixed for
  that long. The ring keeps showing the last frame when the host stops writing (OBSERVED 2026-10-27, section 10).

### 4.2 Captured example

**CAPTURED (2026-10-30, 05:17:33.382)**, the first frame of the capture, L-Connect's Rainbow at brightness 100 %:

```
11 00 00 00 e4 00 14 ca 00 2e b0 00 48 96 00 62 7c 00 7c 62 00 96 48 00 b0 2e 00 ca 14 00 e4 00 00 fe 00 14 e4 00 2e ca 00 48 b0 00 62 96 00 7c 7c 00 96 62 00 b0 48 00 ca 2e 00 e4 14 00 fe 00
11 14 00 00 14 e4 00 2e ca 00 48 b0 00 62 96 00 7c 7c 00 96 62 00 b0 48 00 ca 2e 00 e4 14 00 fe 00 00 e4 00 14 ca 00 2e b0 00 48 96 00 62 7c 00 7c 62 00 96 48 00 b0 2e 00 ca 14 00 e4 00 00 fe
11 28 00 00 00 14 e4 00 2e ca 00 48 b0 00 62 96 00 7c 7c 00 96 62 00 b0 48 00 ca 2e 00 e4 14 00 fe 00 14 e4 00 2e ca 00 48 b0 00 62 96 00 7c 7c 00 96 62 00 b0 48 00 ca 2e 00 e4 14 00 fe 00 00
```

Decoded: two identical 30-LED colour wheels, LED 0 = `e4 00 14`, LED 29 = `fe 00 00`, LED 30 = LED 0 again. The
largest channel value anywhere in the capture is `0xFE` (254), not 255; that matches the `(c × 255) >> 8` scaling
of lian-li-linux's renderer at full brightness (section 7.4).

## 5. `0x10` GetVersion

Request: 64 bytes, `[0] = 0x10`, the rest `00`. Reply on IN `0x81`, 64 bytes:

| Offset | Size | Field | Value on the captured unit |
|---|---|---|---|
| 0 | 1 | opcode echo | `0x10` |
| 1 | 1 | version major | `0x01` |
| 2 | 1 | version minor | `0x03` |
| 3 | 61 | padding | all `00` |

- **Captured exchange (CAPTURED, 2026-10-30).** Request `10 00 .. 00` at 05:19:14.878, with an IN read posted at the
  same moment; reply `10 01 03 00 .. 00` (64 bytes) completed at 05:19:15.080, about 0.2 s later. Third-party code
  got the same 64 bytes on 2026-10-27, logged within a few milliseconds of its descriptor read (OBSERVED).
- **When the vendor sends it (CAPTURED).** Once, 11 ms after L-Connect re-initialised the LCD on its return from
  monitor mode. The capture started with L-Connect already running, so whether it also queries the ring when its
  service starts was not seen (UNKNOWN; likely, since L-Connect shows the ring's firmware version).
- **The version string.** L-Connect formats the version as `lianliLed88_<major>_<minor>` and checks it against
  `^lianliLed88_(\d{1,3})_(\d{1,3})$` (METADATA: the prefix `lianliLed88_` and the pattern are user strings of
  `lianli.lcd207.dll` and `L-Connect.Core.dll`). Two decimal numbers of up to three digits are what two bytes print
  as. The unit above is `lianliLed88_1_3`. That the numbers come from `[1]` and `[2]` was first **PUBLISHED** by
  lianli-python-lcd-overlay; the captured reply `10 01 03` is consistent with it (INFERRED: the string itself is
  never on the wire).
- **Validation.** Accept a reply only when it is at least 3 bytes long, `[0] = 0x10` and `[1] > 0` (the rule
  lianli-python-lcd-overlay applies, **PUBLISHED**). Drain stale IN data before sending the query (INFERRED, hygiene;
  third-party code did this on 2026-10-27 and got a clean reply, OBSERVED).
- Sibling Lian Li MCUs answer `0x10` with other layouts (ASCII strings, a different packet size; **PUBLISHED**,
  lian-li-linux). Do not reuse their parsers for the ring.

## 6. Commands that must not be sent

Only `0x10` and `0x11` are known to be safe. Treat every other first byte as forbidden, and enforce it in code so no
path can emit another opcode. The values below are listed so that nobody tries them.

**The ring's own vendor class** (METADATA, `lcd207.WinUsbLed+USB_CMD` in `lianli.lcd207.dll` 1.4.0.125):

| Value | Vendor name | Why not |
|---|---|---|
| `0x0E` | `Usb_SendRfLong` | RF-dongle name; effect on the ring UNKNOWN |
| `0x12` | `USB_SendHeart` | RF-dongle name; effect UNKNOWN |
| `0x15` | `USB_ResetAnother` | possibly a reset |
| `0x20` | `Usb_FwErase` | **firmware erase** |
| `0x21` | `Usb_FwUpload` | **firmware write** |
| `0x22` | `Usb_FwVerify` | part of the firmware update |
| `0x23` | `Usb_FwReboot` | reboot, possibly into a boot loader |
| `0x25` | `Usb_FwGetInfo` | part of the firmware update; not needed for identification |

**Firmware update.** L-Connect contains a lighting-firmware update path for this ring (METADATA: the string
`StartLightingFirmwareUpdate`, requests `StartFirmwareUpdate` and `CheckFirmwareUpdateStatus`, a status enum with
`None, Ready, Updating, Success, Failed`, and the `UpdateLed` / `CheckLedUpdate` methods). Its wire sequence is
UNKNOWN and is deliberately not described. Never send any of the values above, never replay an update, and do not
claim the ring while L-Connect or its updater runs: during an update the device may reboot and re-enumerate, and a
second writer could interrupt it (INFERRED).

**Values from sibling Lian Li controllers.** None is known to work on the ring, and on related MCUs they reach flash
or boot-loader code (**PUBLISHED**, lian-li-linux; METADATA for the HydroShift II and wired-receiver enums):

| Values | Meaning on the sibling |
|---|---|
| `0x14` | motherboard ARGB sync (wired receivers) |
| `0x15` | save or clear configuration in non-volatile memory (wired receivers) |
| `0x18`, `0x19` | send and apply a compressed lighting package stored in flash (wired receivers) |
| `0x13`, `0x16`, `0x17`, `0x26`, `0x27`, `0x29` | fans, groups, LCD reboot, themes (wired receivers) |
| `0x50`, `0x51`, `0x60`, `0x61`, `0x62`, `0x64` | motor, temperature, pump and motherboard sync (HydroShift II LED MCU `0x8051`) |
| `0x72`; `0x80` to `0x83` | jump to the in-application programmer; firmware update, erase, verify, end (HydroShift II) |

**LCD commands.** The LCD (`1CBE:A088`) is a different device with its own encrypted header. Its command byte `0x11`
inside that header means something else entirely (GetH264Block; **PUBLISHED**, hypercolor; METADATA,
`lcd207.CmdType`). Never send LCD commands to the ring or ring packets to the LCD.

## 7. Sequences

### 7.1 Initialisation

There is no handshake. The vendor streams frames without any preceding command (CAPTURED: frames from the first
millisecond of the capture; the only `0x10` came later), and lian-li-linux and OpenRGB open the device and start
writing (**PUBLISHED**). A minimal sequence that third-party code used on 2026-10-27 (OBSERVED):

1. Enumerate `0416:8050` exactly (VID **and** PID). Open interface 0.
2. Optionally check the endpoints: bulk OUT `0x01` and bulk IN `0x81`, 64 bytes.
3. Optionally drain IN `0x81`, send `0x10`, read up to 64 bytes with a timeout of a few hundred milliseconds, and
   tolerate no reply (INFERRED: the firmware answers, but nothing depends on it).
4. Write nothing until there is a frame to show. Until then the ring keeps whatever the previous host left
   (OBSERVED 2026-10-27; section 10).

### 7.2 One frame

```
for chunk in 0, 1, 2:
    packet = 64 zero bytes
    packet[0] = 0x11
    packet[1] = chunk * 20
    for i in 0 .. 19:
        packet[4 + 3*i], packet[5 + 3*i], packet[6 + 3*i] = R, G, B of LED (chunk*20 + i)
    bulk write packet to endpoint 0x01
```

Send all three packets of a frame together and do not let a second writer interleave its packets with them; a mix of
two frames would show as mismatched 20-LED blocks (INFERRED from the chunked layout). Treat a short or failed write
as a failed frame and resend the whole frame (INFERRED).

- **Off** is a frame of 60 × `00 00 00`. The vendor has no off control for the ring; its brightness at 0 % sends
  exactly that (CAPTURED, 05:25:21).
- **Static colour** is the same colour in all 60 slots (OBSERVED 2026-10-27 with red, green, blue, white).

### 7.3 Streaming rate

| Who | Rate | Tag |
|---|---|---|
| L-Connect, Rainbow (brightness 100 %, speed 75) | median 50 ms between frames, mean 48 ms (about 20.7 frames/s); most gaps 40 to 55 ms, longest 225 ms | CAPTURED |
| L-Connect, Wave (from about 05:22:30) | about 22 frames/s | CAPTURED |
| L-Connect over the whole capture | 15 095 frames in 11.6 min, 20 to 22 frames/s, no gap over 1 s, including the panel's trip to monitor mode and back (05:18:27 to 05:19:12, 2 970 packets, all completed) | CAPTURED |
| Third-party code | frame intervals of 27.5 ms (about 36 frames/s) and 41.25 ms, no failed frame | OBSERVED 2026-10-27 |
| Third-party code, 20 s at each requested rate | 60 requested: 57 delivered; 120: 111; 240: 206 frames/s; every frame accepted, no tearing seen; the three writes of a frame 1.0 to 1.3 ms on average | OBSERVED 2026-11-03 |
| lian-li-linux, direct mode | sleeps 16 ms between passes, a little under 60 Hz | PUBLISHED, lian-li-linux |

The ring sustains at least 206 frames/s (OBSERVED 2026-11-03): third-party code streamed a 30-frame rainbow at
requested rates of 60, 120 and 240 frames/s for 20 s each and delivered 57, 111 and 206 frames/s, with every frame
accepted (no short or failed write) and no tearing or stutter visible to the observer. The three 64-byte writes of a
frame took 1.0 to 1.3 ms on average (single frames up to 73 ms, from host scheduling), so the rate the ring itself
allows is higher than anything measured; the limit reached was the host loop's. A frame is 192 bytes, so 240 frames/s
is about 46 KB/s, far below what a full-speed bulk pipe carries. The cadence stayed at 20 to 22 frames/s through
L-Connect's speed and effect changes (CAPTURED), so its speed setting changes the animation's step per frame, not the
frame rate. Host note: a wait between frames that is rounded to the Windows timer tick (15.6 ms for a background
process) halves a 60 frames/s stream to 33 (OBSERVED 2026-11-03); a high-resolution timer is needed above about
30 frames/s.

### 7.4 How the vendor's effects are produced

**All of them are rendered on the PC and streamed** (CAPTURED). Every L-Connect setting the owner changed (the
effect from Rainbow to Wave, the speed from 75 to 25, the brightness, brightness 0 %) changed only the RGB bytes of
the `0x11` frames; no other command appeared. L-Connect has no off control for the ring (its lighting on/off icon
did nothing on the wire). The ring therefore needs no effect support of its own, and none is known.

**Rainbow is byte-identical to lian-li-linux's renderer (CAPTURED).** During the first four minutes of the capture
(Rainbow, brightness 100 %, speed 75) L-Connect sent exactly 30 distinct frames, over and over. Each is the frame in
section 4.2 rotated by a whole number of LEDs, and each frame is the previous one shifted by one LED towards higher
LED indices (`frame[n+1][i] = frame[n][i-1]`). Frame 0 is identical to the Rainbow frame 0 that lian-li-linux's
renderer produces at brightness level 4, clockwise (test vector TV4, section 12). The other effects were not
compared: Wave (72 distinct frames in each 20 s window) does not obviously match, so whether all 19 match is
UNKNOWN.

**The effect catalogue.** The vendor enum gives the names and numbers (METADATA, `UniversalScreen8p8InchLightingMode`).
lian-li-linux reimplemented all 19 renderers and states that they follow the vendor's own algorithms (**PUBLISHED**,
`crates/lianli-media/src/rgb/universal/`, with golden-hash tests for every effect; only Rainbow was re-checked here):

| Value | Name | Frames (60 LEDs) | Base interval (hundredths of a tick) | Colours | Direction |
|---|---|---|---|---|---|
| 0 | Rainbow | 30 | 1650 | none | yes |
| 1 | Wave | 48 per colour | 1430 | 1-6 | yes |
| 2 | StaticColor | 1 | 1100 | 1 | no |
| 3 | Breathing | 170 | 1100 | 1 | no |
| 4 | RainbowMorph | 127 | 1100 | none | no |
| 5 | Paint | 31 per colour | 1100 | 1-6 | no |
| 6 | Runway | 70 | 1100 | 2 (background, moving block) | no |
| 7 | Tide | 16 per colour | 1100 | 1-6 | no |
| 8 | BlowUp | 51 per colour | 1100 | 1-6 | no |
| 9 | Meteor | 30 per colour | 1100 | 1-6 | yes |
| 10 | Snooker | 28 per colour | 1100 | 1-6 | no |
| 11 | Mixing | 32 | 1100 | 2 | no |
| 12 | PingPong | 58 per colour | 1100 | 1-6 | no |
| 13 | BulletStack | 325 | 1100 | none (fixed table) | yes |
| 14 | Twinkle | 200 | 1100 | none (fixed table, seeded random) | no |
| 15 | River | 6 | 1430 | 2 | yes |
| 16 | Hourglass | 70 per colour | 1100 | 1-4 | no |
| 17 | ElectricCurrent | 45 per colour | 1100 | 1-4 | no |
| 18 | RainbowWave | 60 | 1650 | none | yes |

Parameters, all host-side (**PUBLISHED**, lian-li-linux):

- **Speed**: five levels; frame interval = base × `[7, 6, 5, 4, 3][level]` hundredths of a 0.625 ms tick (level 0 is
  the slowest). Rainbow at level 3 is 1650 × 4 = 6600 hundredths = 41.25 ms.
- **Brightness**: five levels with factors `[0, 64, 128, 192, 255]`, applied as `(c × f) >> 8`, so full brightness
  maps 255 to 254 (consistent with the capture's peak of `0xFE`, CAPTURED).
- **Direction**: the vendor's `IsReverseDirection` (METADATA) selects the reverse order on the effects marked "yes".

How L-Connect's 0-100 sliders map to the five levels is UNKNOWN; dividing by 25 is a guess (INFERRED). The captured
Rainbow at "speed 75" advanced one frame every 48 to 50 ms, slower than the 41.25 ms the formula gives for level 3,
so L-Connect's own timing differs from lian-li-linux's (CAPTURED for the rate; the reason is UNKNOWN).

## 8. LED topology

The wire order is simply LEDs 0 to 59. Where those LEDs sit on the bar has to be known for anything positional
(screen-edge colour sync, effects that should be symmetric).

**What the effect tables assume (PUBLISHED, lian-li-linux).** The published renderers treat the ring as a closed
loop with mirror symmetry about LEDs 17 and 47: LEDs 44 to 50 and 14 to 20 act as two 7-LED cells centred on 47 and
17, joined by two 23-LED arms (51 to 59 plus 0 to 13, and 21 to 43). lianli-python-lcd-overlay instead spaces the
60 LEDs evenly round the 480 x 1920 perimeter starting at the bottom-left corner (**PUBLISHED**); that spacing does
not match the unit below.

**What one unit showed (OBSERVED, 2026-11-02, screen mounted in landscape with the long sides horizontal, panel in
its landscape view, seen from the front).** Single white LEDs were lit by wire index, one at a time, by third-party
code:

| Wire LED lit | Seen at |
|---|---|
| 0 | the bottom long side, left of its centre |
| 1 | one step to the right of LED 0: **indices run counterclockwise** |
| 55 | the bottom long side, nearer the left corner than the centre |
| 52 | the bottom long side, close to the left corner |
| 51 | the last LED of the bottom long side at the left corner |
| 50 | the first LED of the left short side, just above that corner |
| 17 | the centre of the right short side |
| 47 | the centre of the left short side |

This is exactly the 7/23 split the published tables imply: the bottom long side is wire LEDs 51 to 59 and 0 to 13
(23 LEDs, centre LED 2, so LED 0 sits two LEDs left of the centre), the right short side 14 to 20 (centre 17), the
top long side 21 to 43 (centre 32) and the left short side 44 to 50 (centre 47). The corners at 13/14, 20/21 and
43/44 follow from the split (INFERRED; only the 50/51 corner was checked). The published effect tables' mirror axes
therefore fall on the centres of the short sides on this unit.

An earlier session on the same unit (2026-10-27) recorded LED 0 "at the bottom right" and placed the short-side
centres at wire LEDs 26 and 56; the 2026-11-02 check, which lit the wire indices directly, supersedes it (the earlier
notes appear to mix the bar's own indices with a trial mapping of a host-side model).

**Direction of the vendor's Rainbow.** L-Connect's Rainbow with "reverse direction" off moves towards higher wire
indices (CAPTURED, section 7.4), which on this unit is counterclockwise seen from the front (INFERRED, combining the
capture with step 2).

Whether every unit is wired the same way, and how L-Connect's "rotate 180" setting relates to the LED order, is
UNKNOWN; only one unit was seen.

## 9. Recovery

- **After a sleep or a re-plug, reopen.** After an S3 sleep, third-party code closed its handle, reopened the ring and
  had it ready for frames again about 5 s after the resume (OBSERVED, 2026-10-30); after an overnight sleep
  (2026-11-02) it reconnected the same way without help (OBSERVED). No command beyond the usual `0x10` and `0x11`
  was needed (OBSERVED).
- **A stalled pipe:** clear the halt (`WinUsb_ResetPipe`, or libusb's clear-halt) and resend the whole frame once
  (lian-li-linux clears halts on both endpoints and retries, **PUBLISHED**; resending the whole frame is INFERRED).
- **Never reset the USB port or the hub.** The ring and the LCD share the screen's internal hub; lian-li-linux avoids
  port resets because they take down sibling devices like this one (**PUBLISHED**) and recovers by close and reopen.
- **Reopen the same unit.** lian-li-linux and OpenRGB reopen the first `0416:8050` they find (**PUBLISHED**); with two
  screens, reopen by the saved device path instead (INFERRED).
- **Repeated re-enumeration** can mean a firmware update in progress (section 6). Back off instead of reclaiming
  (INFERRED).

## 10. Standalone behaviour

- **The ring holds the last frame.** After third-party code wrote a colour and exited, the ring kept it (OBSERVED,
  2026-10-27). lianli-python-lcd-overlay's one-shot colour command relies on the same thing (**PUBLISHED**). No
  keep-alive was needed in those tests (OBSERVED); how long a frame stays without any host was not measured
  (UNKNOWN).
- **Animations freeze when the host stops.** Every effect is a host stream (section 7.4), so the ring shows the last
  frame it received (INFERRED). A host that stops between the three packets of a frame leaves a mix of two frames
  (INFERRED).
- **No on-device effects, no save.** No effect upload, effect selection or save command appeared in the capture
  (CAPTURED), OpenRGB marks the ring as having neither effects nor save (**PUBLISHED**), and the ring's vendor class
  declares none (METADATA, section 6, which lists only RF-style and firmware values besides `0x10`/`0x11`).
  Whether the firmware has a hidden mode is UNKNOWN; nothing a host sends with `0x11` is expected to reach flash
  (INFERRED).
- **Power-on with no host.** What the ring shows after a power cycle before any host writes is UNKNOWN. On the bench
  the vendor's software or third-party code took the ring over within seconds of every boot, so it was never watched
  idle (OBSERVED, 2026-10-28 and 2026-10-31).
- **During sleep.** Whether the ring stays lit in S3 is UNKNOWN. (The LCD panel on the same hub stayed powered in S3
  on this PC and drew its own clock; see the LCD document.)
- **When the screen is in monitor mode.** The ring stays on the bus and keeps accepting frames while the LCD
  re-enumerates as `1A86:AD21` and back (CAPTURED: L-Connect's frames continued and completed throughout
  05:18:27 to 05:19:12; consistent with a report in lian-li-linux's issue tracker, **PUBLISHED**).

## 11. Hazards

| Hazard | Guard |
|---|---|
| Wrong device: VID `0x0416` is shared with other Lian Li controllers (section 1.3), and `0x8051` uses the same opcodes with other packet sizes | Match VID **and** PID `0x8050` exactly; never reuse another device's packet builder |
| Unknown opcodes: the ring's own vendor class declares firmware erase, write, verify and reboot values; siblings have flash, NVRAM and boot-loader commands | Allow only `0x10` and `0x11` as `[0]`, in the one place every packet passes |
| Interrupting a vendor firmware update | Do not open the ring while the vendor software or its updater runs; back off on repeated re-enumeration |
| Malformed `0x11` | Always 64-byte transfers, offsets 0/20/40, bytes 2-3 zero; anything else is an untested experiment |
| Two writers (vendor service, OpenRGB, a second driver) | WinUSB is exclusive; report the device busy instead of stopping other software; never interleave packets of two frames |
| A hung pipe stalling the host | Per-pipe timeouts; drop stale frames rather than queue them |
| Stale IN data corrupting the version read | Drain IN before `0x10`; validate echo and major |
| Ring and LCD confused (the LCD's `0x11` is a different command inside an encrypted header) | Separate code and separate device matches for the two |
| Port or hub resets taking the LCD down with the ring | Recover by close and reopen only |
| Power and heat: 60 LEDs at full white, power budget UNKNOWN (the MCU declares 70 mA; the LEDs' supply is not described) | Consider a cap on R+G+B per LED; the vendor's own effects peak at 254 per channel and its Mixing effect scales bright mixes down (**PUBLISHED**, lian-li-linux) |
| Photosensitivity: whole-ring flashing at high frame rates | Limit full-ring flashes to 3 per second unless the user opts in (INFERRED, general practice) |

## 12. Test vectors

All packets are 64 bytes, shown in full.

**TV1, index frame** (built from the layout in 4.1): LED `i` = `(i, i+1, i+2)` for i = 0..59.

```
11 00 00 00 00 01 02 01 02 03 02 03 04 03 04 05 04 05 06 05 06 07 06 07 08 07 08 09 08 09 0a 09 0a 0b 0a 0b 0c 0b 0c 0d 0c 0d 0e 0d 0e 0f 0e 0f 10 0f 10 11 10 11 12 11 12 13 12 13 14 13 14 15
11 14 00 00 14 15 16 15 16 17 16 17 18 17 18 19 18 19 1a 19 1a 1b 1a 1b 1c 1b 1c 1d 1c 1d 1e 1d 1e 1f 1e 1f 20 1f 20 21 20 21 22 21 22 23 22 23 24 23 24 25 24 25 26 25 26 27 26 27 28 27 28 29
11 28 00 00 28 29 2a 29 2a 2b 2a 2b 2c 2b 2c 2d 2c 2d 2e 2d 2e 2f 2e 2f 30 2f 30 31 30 31 32 31 32 33 32 33 34 33 34 35 34 35 36 35 36 37 36 37 38 37 38 39 38 39 3a 39 3a 3b 3a 3b 3c 3b 3c 3d
```

**TV2, off**: `11 00 00 00`, `11 14 00 00`, `11 28 00 00`, each followed by 60 × `00` (the same bytes L-Connect sends
at brightness 0 %, CAPTURED, 05:25:21.537).

**TV3, static red**: `11 00 00 00`, `11 14 00 00`, `11 28 00 00`, each followed by 20 × `ff 00 00` (built from the
layout; OBSERVED 2026-10-27 to light the whole ring red). Through lian-li-linux's renderer at full brightness the
same colour is `fe 00 00` per LED (**PUBLISHED**).

**TV4, Rainbow frame 0** at brightness level 4, forward direction: the three packets of section 4.2. **CAPTURED**
(05:17:33.382) from L-Connect and identical to lian-li-linux's renderer output (**PUBLISHED**). The next captured
frame is the same pattern moved one LED towards higher indices; 30 frames make one cycle.

**TV5, GetVersion**:

- Request: `10` followed by 63 × `00` (CAPTURED, 05:19:14.878).
- Reply: `10 01 03` followed by 61 × `00` (CAPTURED, 05:19:15.080), which formats as `lianliLed88_1_3` and matches
  `^lianliLed88_(\d{1,3})_(\d{1,3})$`. The extremes `10 FF 00` and `10 01 FF` format as `lianliLed88_255_0` and
  `lianliLed88_1_255`, which also match.
- Reject: `11 01 03` (wrong echo), `10 00 03` (major 0), and any reply shorter than 3 bytes.

**TV6, negative test**: a builder for this ring must never produce a packet with a first byte other than `0x10` or
`0x11`, an offset other than 0, 20 or 40, or 15-LED chunks (the `0x8051` layout).

lian-li-linux's renderer tests pin a hash of every frame of every effect at all brightness levels and both
directions (**PUBLISHED**, `crates/lianli-media/src/rgb/universal/mod.rs`); a port of the effects can check itself
against those.

## 13. Open items

| # | Question | How to settle |
|---|---|---|
| 1 | ~~Do the 20-LED packets latch one by one, or only after the packet at offset 40?~~ **Settled 2026-11-03: one by one** (section 4.1) | done: lone offset-0, offset-20 and offset-40 packets each took effect at once |
| 2 | Meaning of bytes 2 and 3 of `0x11` on this firmware (count and end-of-frame flag, as on the wired receivers?) | one packet at a time with non-zero values, after item 1 |
| 3 | Offsets that are not multiples of 20, or above 40 | as item 2 |
| 4 | ~~The highest frame rate the ring sustains~~ **At least 206 frames/s (2026-11-03, section 7.3)**; the true ceiling is still open | a host loop faster than 206 frames/s, or a bulk write that is not awaited per packet |
| 5 | Whether every unit is wired the same way (one unit seen; section 8 settled it for that unit on 2026-11-02) | the same single-LED check on another unit |
| 6 | What the ring shows at power-on with no host, and during S3 sleep | power-cycle with no host software running; sleep with nothing claiming the ring |
| 7 | Whether L-Connect also sends `0x10` when its service starts | a capture that starts before the vendor service |
| 8 | Whether the other 18 vendor effects match lian-li-linux's frames as Rainbow does | compare captured frames of each effect against the published renderer |
| 9 | How L-Connect maps its 0-100 speed and brightness sliders to frames | capture each slider position |
| 10 | The firmware-update sequence and boot-loader identity | only passively, if an owner runs a vendor update on their own; needed only to recognise it and stay away. Never trigger or replay it |
| 11 | The ring MCU's silicon | a look at the board |
