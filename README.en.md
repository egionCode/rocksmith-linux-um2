# Rocksmith 2014 on Linux with a Behringer UM2

[Português](README.md) | English

This is how I got Rocksmith 2014 Remastered running on CachyOS with a Behringer UM2 audio interface instead of the Real Tone Cable. Low latency, no noise, CDLC and RSMods working.

It took a whole afternoon and about four failed attempts. This README exists so you don't have to make the same mistakes.

## Quick summary

If you just want to know what works:

| Piece | What I used |
|---|---|
| Distro | CachyOS (Arch based), PipeWire 1.6.9 |
| Compatibility layer | GE-Proton11-7 |
| Audio driver | PipeASIO 1.10.0 (`pipeasio` or `pipeasio-bin` from the AUR) |
| ASIO bridge into the game | RS_ASIO 0.7.5 |
| Interface | Behringer UM2 (TI PCM2902 chip) |
| CDLC | RSCDLCEnabler, old 130 KB build |
| Mods | RSMods 1.2.8.4 (Fast Load and auto-enter profile) |

Final Steam launch options:

```
PROTON_USE_WOW64=1 WINEDLLPATH=<path printed by pipeasio-manage> WINEDLLOVERRIDES="D3DX9_42=n,b;xinput1_3=n,b" %command%
```

The audio path looks like this:

```
Guitar -> UM2 -> PipeWire -> PipeASIO -> RS_ASIO -> Rocksmith
```

No JACK, no hand-compiled wineasio.

## Contents

1. [Before you start](#before-you-start)
2. [Basic setup](#basic-setup)
3. [CDLC](#cdlc)
4. [RSMods](#rsmods)
5. [Advanced](#advanced)
6. [Problems I ran into](#problems-i-ran-into)
7. [Sources](#sources)

## Before you start

You'll need:

- Rocksmith 2014 Edition Remastered on Steam
- The free DLC "Smashing Pumpkins - Cherub Rock" (CDLC depends on it)
- An AUR helper (`paru` or `yay`)
- `protontricks` (it's in the CachyOS repos)
- Your Steam library on a Linux filesystem (ext4, btrfs). NTFS causes trouble with Proton prefixes.

Tested on 2026-10-06 with kernel 7.2, PipeWire 1.6.9 and GE-Proton11-7. If you're reading this much later, versions have probably moved on and some step may not match anymore.

## Basic setup

### 1. Install PipeASIO

```bash
paru -S pipeasio-bin
```

`pipeasio-bin` is prebuilt. The plain `pipeasio` package builds from source and pulls in `mingw-w64-gcc`, which takes a while. Both install the same thing.

### 2. Install GE-Proton 11

Rocksmith is a 32-bit game. For the 32-bit PipeASIO to load, Proton has to run in "new WoW64" mode, which you turn on with `PROTON_USE_WOW64=1`.

The catch: Valve's Proton 11.0-100 ignores that variable. GE-Proton honors it. Hence GE.

With ProtonUp-Qt, just pick GE-Proton11-7. By hand:

```bash
cd "$(mktemp -d)"
N=GE-Proton11-7-x86_64
U=https://github.com/GloriousEggroll/proton-ge-custom/releases/download/GE-Proton11-7
curl -fLO "$U/$N.tar.gz"
curl -fLO "$U/$N.sha512sum"
sha512sum -c "$N.sha512sum"
tar -xzf "$N.tar.gz" -C ~/.local/share/Steam/compatibilitytools.d/
```

Watch the file name: starting with 11-4 the release files got an `-x86_64` suffix. Older guides use the name without it, and the download comes back as a 9-byte "Not Found".

Restart Steam afterwards so it picks up the new Proton.

### 3. Set up the game in Steam

In Rocksmith 2014 > Properties:

- Compatibility: tick "Force" and pick GE-Proton11-7
- Launch the game once and close it. This creates the prefix at `<SteamLibrary>/steamapps/compatdata/221680/pfx`.

### 4. Register PipeASIO in the game's prefix

PipeASIO ships with a manager that finds Steam games on its own:

```bash
# lists games and shows each one's id
pipeasio-manage list

# find the block with "appid": "221680" and copy its "id" (something like steam:e474...)
pipeasio-manage --target steam:YOUR_ID --include-32 install
```

`--include-32` is required: without it only the 64-bit driver gets registered, and Rocksmith doesn't use that one.

At the end it prints the launch options line with the right `WINEDLLPATH` for your machine. Use exactly what it shows.

If you ever delete the prefix or switch Proton versions, run this again.

### 5. Install RS_ASIO

Grab version 0.7.5 from [github.com/mdias/rs_asio/releases](https://github.com/mdias/rs_asio/releases) and copy `RS_ASIO.dll`, `avrt.dll` and `RS_ASIO.ini` into the game folder (where `Rocksmith2014.exe` lives).

Replace the contents of `RS_ASIO.ini` with this:

```ini
[Config]
EnableWasapiOutputs=0
EnableWasapiInputs=0
EnableAsio=1

[Asio]
BufferSizeMode=driver
CustomBufferSize=

[Asio.Output]
Driver=PipeASIO
BaseChannel=0
AltBaseChannel=
EnableSoftwareEndpointVolumeControl=1
EnableSoftwareMasterVolumeControl=1
SoftwareMasterVolumePercent=100
EnableRefCountHack=

; guitar. Channel=0 is the left (L) side of the UM2.
; see "Finding which channel the guitar comes in on" in the advanced section
[Asio.Input.0]
Driver=PipeASIO
Channel=0
EnableSoftwareEndpointVolumeControl=1
EnableSoftwareMasterVolumeControl=1
SoftwareMasterVolumePercent=100
EnableRefCountHack=

; disabled on purpose: with an empty Driver the game doesn't see a phantom second player
[Asio.Input.1]
Driver=
Channel=1

[Asio.Input.Mic]
Driver=
Channel=1
```

### 6. Check Rocksmith.ini

In the same folder, make sure the `[Audio]` section has:

```ini
ExclusiveMode=1
Win32UltraLowLatencyMode=1
```

Guides that use plain ALSA tell you to set both to `0`. With RS_ASIO it's the other way around.

### 7. Point PipeASIO at the UM2

Open `pipeasio-settings` (the GUI that comes with the package) and pick the UM2 as the input device. Or edit `~/.config/pipeasio/config.ini` directly:

```ini
[pipeasio]
inputs = 2
outputs = 2
buffer_size = 128
fixed_buffer_size = 1
sample_rate = 48000
auto_connect = 1
input_device = alsa_input.usb-Burr-Brown_from_TI_USB_Audio_CODEC-00.analog-stereo-input
```

PipeWire calls the UM2 "Burr-Brown from TI USB Audio CODEC", not "Behringer". To check yours:

```bash
pw-cli ls Node | grep node.name | grep -i burr
```

The PipeASIO guide for the Real Tone Cable says to use `inputs = 1` because the cable is mono. The UM2 is stereo, so leave it at 2 and pick the channel in `RS_ASIO.ini`.

### 8. Launch options

In Properties > General > Launch Options:

```
PROTON_USE_WOW64=1 WINEDLLPATH=<path from step 4> %command%
```

Start the game and play something. If the guitar shows up and you can hear it, the hard part is over.

## CDLC

### The enabler that works

CDLC needs a modified `D3DX9_42.dll` in the game folder. Here's the trap: recent CustomsForge enabler builds crash the game on Linux, about 7 seconds after launch. I tried the "TooManyCoresFix" variant (248 KB) and it crashed under GE-Proton11.

What worked was an old RSCDLCEnabler build, 130 KB, posted in the [devroom.io](https://www.devroom.io/2025/12/02/rocksmith-2014-on-cachyos-arch-linux/) guide. Download it from there and drop it in the game folder.

SHA-256 of the file I used:

```
c09dacbd428a5b55db814b46401224fd7ea9c295057112334eb50c1768d3c776
```

It's a third-party binary with no official hash to check against. Its internal strings match the RSCDLCEnabler project, but whether you trust it is up to you.

This old build needs the Cherub Rock DLC installed.

### Launch option

Add the override so Proton loads the DLL from the game folder:

```
WINEDLLOVERRIDES="D3DX9_42=n,b"
```

### Songs

Download from [CustomsForge Ignition](https://ignition4.customsforge.com/). Always grab the files ending in `_p.psarc` (PC). The `_m.psarc` ones are for Mac.

Put them in `Rocksmith2014/dlc/`. Subfolders like `dlc/cdlc/` work too if you want to keep them apart from official DLC.

## RSMods

RSMods skips the intros, drops you straight into your profile, and has a pile of other options. It officially supports Wine/Proton.

### Install

First, back up your save. It lives outside the prefix:

```bash
cp -a ~/.local/share/Steam/userdata/<YOUR_STEAM_ID>/221680/remote ~/rocksmith-save-backup
```

Download `RS2014-Mod-Installer.exe` from [github.com/Lovrom8/RSMods/releases](https://github.com/Lovrom8/RSMods/releases) and run it inside the game's prefix:

```bash
PROTON_USE_WOW64=1 protontricks-launch --appid 221680 ~/Downloads/RS2014-Mod-Installer.exe
```

When it asks for the save folder, use the Wine path:

```
Z:\home\<your_user>\.local\share\Steam\userdata\<YOUR_STEAM_ID>\221680\remote
```

`Z:` is the Linux root as seen from inside Wine.

To open RSMods again later:

```bash
cd <SteamLibrary>/steamapps/common/Rocksmith2014/RSMods
PROTON_USE_WOW64=1 protontricks-launch --cwd-app --appid 221680 ./RSMods.exe
```

### The mods I use

- Fast Load (Set and Forget tab): skips the startup screens. It edits `cache.psarc` once and keeps the original as `cache.bak`. Works even without the RSMods DLL loaded.
- Force Profile Load: presses Enter for you and loads your last profile. This one needs the RSMods DLL.

### Launch option

The RSMods DLL is `xinput1_3.dll`. Without an override Proton uses its own built-in version and the in-game mods never run:

```
WINEDLLOVERRIDES="D3DX9_42=n,b;xinput1_3=n,b"
```

Turn on one mod at a time and test. If the game breaks, you'll know which one did it.

## Advanced

### How the pieces fit together

Rocksmith was built to talk to the Real Tone Cable through WASAPI on Windows. On Linux that turns into a chain:

1. The UM2 shows up in PipeWire as a regular stereo device.
2. PipeASIO is an ASIO driver for Wine that talks to PipeWire directly. It has two halves: a Windows DLL (`pipeasio32.dll`) and a Linux library (`pipeasio32.so`).
3. RS_ASIO injects itself into Rocksmith and swaps the WASAPI devices the game looks for with fake devices backed by an ASIO driver.
4. The game thinks it's talking to the official cable.

The fragile bit is step 2 in 32-bit. The 32-bit Windows DLL has to call into the 64-bit Linux library, and that only works under Wine's new WoW64. That's what `PROTON_USE_WOW64=1` is for. `WINEDLLPATH` tells Wine where to find the `.so`.

### Finding which channel the guitar comes in on

I got this wrong. I assumed the guitar in the UM2's instrument input would land on the right channel, but it came in on the left. Don't guess, measure:

```bash
#!/usr/bin/env bash
# records 15 s from the UM2 and prints the level of each channel.
# keep playing the guitar while it records.
# needs: pipewire (pw-record), python3
set -euo pipefail
DIR=$(mktemp -d)
NODE=alsa_input.usb-Burr-Brown_from_TI_USB_Audio_CODEC-00.analog-stereo-input

# pw-record has no duration flag, so kill it after 15 s
pw-record --target "$NODE" --channels 2 --rate 48000 --format s16 "$DIR/t.wav" &
sleep 15
kill $!

python3 -I - "$DIR/t.wav" <<'EOF'
import sys, wave, array, math
w = wave.open(sys.argv[1]); n = w.getnchannels()
a = array.array('h', w.readframes(w.getnframes()))
for c in range(n):
    s = a[c::n]
    rms = math.sqrt(sum(x*x for x in s) / max(len(s), 1))
    print(f"channel {c} ({'L' if c == 0 else 'R'}): peak={max(abs(x) for x in s)/327.67:.1f}%  rms={rms/327.67:.2f}%")
EOF
rm -rf "$DIR"
```

Whichever channel shows signal is the number that goes in `Channel=` under `[Asio.Input.0]`.

### Reading the RS_ASIO log

RS_ASIO writes `RS_ASIO-log.txt` to the game folder on every run. It's the first place to look.

When it works, you'll see something like:

```
AsioHelpers::FindDrivers
  PipeASIO
ASIO input channels info:
  0 - ... name: in_1
  1 - ... name: in_2
{ASIO IN 0} ... ::Start
```

The errors I hit and what they mean:

| Log message | Cause |
|---|---|
| `cannot load pipeasio32.so: needs new WoW64` | `PROTON_USE_WOW64=1` is missing, or Proton is ignoring it (Valve 11.0-100) |
| `FindDrivers` with no `PipeASIO` under it | Driver not registered in the prefix. Run `pipeasio-manage install` |
| Log doesn't get updated | The game crashed before RS_ASIO loaded. Usually the `D3DX9_42.dll` |

If the log isn't even updated, Steam's own log shows how long the process lived:

```bash
grep 221680 ~/.local/share/Steam/logs/console_log.txt | tail
```

### Checking PipeWire links

With the game running:

```bash
pw-link -l | grep -i -A4 pipeasio
```

You should see the UM2 capture ports linked to the PipeASIO inputs. With `auto_connect = 1` this happens on its own.

### Latency

With `buffer_size = 128` at 48 kHz you get about 2.7 ms per buffer, and RS_ASIO confirms it in the log (`actual buffer duration: 2ms (128 frames)`). I didn't notice any lag. If you hear crackling, bump it to 256 in PipeASIO's `config.ini`.

## Problems I ran into

In the order they happened:

1. Launch options without `PROTON_USE_WOW64=1`. The game started with zero audio devices.
2. Used the system's default Proton, which didn't turn on WoW64. Switched to GE-Proton11-7.
3. Fresh prefix with no driver registered. Got "No audio output device is detected". Fixed with `pipeasio-manage install`.
4. Guitar on the wrong channel. Game audio worked, guitar was silent. Measured with `pw-record` and switched to `Channel=0`.
5. New CDLC enabler crashing on startup. Swapped it for the old 130 KB build.
6. The guitar "stopped working" after installing RSMods. The cable had come out of the UM2. Check the obvious stuff first.

One more thing: `pipeasio-register --help` doesn't print help. It registers the driver in your default `~/.wine`. It doesn't break anything, but it's not what you'd expect.

## Sources

- [PipeASIO](https://github.com/M0n7y5/pipeasio)
- [RS_ASIO](https://github.com/mdias/rs_asio)
- [rocksmith-linux-pipeasio-guitar-input-fix](https://github.com/UkrainianCitizen/rocksmith-linux-pipeasio-guitar-input-fix), especially [gotchas.md](https://github.com/UkrainianCitizen/rocksmith-linux-pipeasio-guitar-input-fix/blob/main/docs/gotchas.md)
- [linux-rocksmith (Nizo)](https://codeberg.org/nizo/linux-rocksmith), the classic JACK + wineasio guide
- [Rocksmith 2014 on CachyOS/Arch Linux (devroom.io)](https://www.devroom.io/2025/12/02/rocksmith-2014-on-cachyos-arch-linux/)
- [CustomsForge: CDLC on Steam Proton not working anymore](https://customsforge.com/topic/66510-linux-solved-rsremastered-cdlc-on-steam-proton-not-working-anymore/)
- [RSMods](https://github.com/Lovrom8/RSMods)
- [GE-Proton](https://github.com/GloriousEggroll/proton-ge-custom)
- [Rocksmith 2014 on ProtonDB](https://www.protondb.com/app/221680)
