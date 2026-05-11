# Garmin RINO Burst Decoder

A self-calibrating narrowband burst decoder targeting Garmin RINO position packets transmitted over FRS/GMRS frequencies. Captures live IQ from SDRconnect at 250 ksps, detects and decodes position bursts in real time.

## What it does

The RINO radio transmits a short data burst (~270 ms) on the FRS/GMRS channel containing the sender's GPS position, callsign, altitude, and icon. This program listens continuously, detects those bursts, and decodes the position data.

Detection and decoding is fully automatic — no manual tuning required. On each capture the program measures:

- Carrier offset from the SDR centre frequency
- Signal tone frequencies and decision threshold
- Preamble baud rate
- Data baud rate (derived from preamble via the 25/16 ratio)
- Demodulation LPF bandwidth

## Signal structure

```
[ Preamble ~300ms ] [ Carrier hold ~250ms ] [ Data burst ~270ms ]
```

The preamble is a continuous alternating tone at ~384 baud. The data burst follows at ~600 baud after a brief carrier hold. The decoder finds the preamble first, uses its measured characteristics as a reference to gate the burst search, then decodes the burst using a Gardner/M&M PLL bit sampler with eye-diagram seeding.

## Architecture

```
SDRconnect (250 ksps)
    └─ BurstCapture.cs       — amplitude gate, decimation 2M→250k, burst windowing
         └─ BurstDecoder.cs
              ├─ Cf32Loader          — carrier offset measurement and removal
              ├─ SignalAnalyser      — preamble detection, burst gating, parameter measurement
              ├─ BurstDecoder        — fine demod, eye diagram, M&M tracking, bit sampling
              └─ RinoDecoder         — NRZI decode, frame sync, CRC, position extraction
```

## How to use

### Requirements

The decoder is a standalone self-contained executable — just run it directly, no installation needed. If it doesn't start, install the [.NET 8 Runtime](https://dotnet.microsoft.com/download/dotnet/8.0).

You also need:
- [SDRconnect](https://www.sdrplay.com/sdrconnect/) running with a compatible SDR (RSP1A, RSPdx, etc.)
- The SDRconnect WebSocket server enabled (see below)
- An SDR tuned to the target FRS/GMRS frequency

---

### Enabling the SDRconnect WebSocket

The decoder receives IQ samples from SDRconnect over a WebSocket connection. This must be enabled before starting the decoder:

1. Open SDRconnect
2. Go to **Settings → API / WebSocket**
3. Enable the WebSocket server
4. Set the centre frequency, mode to `NFM`, and sample rate to `250 ksps`
5. Start the SDR streaming

SDRconnect must be running and the WebSocket must be active before you launch the decoder.

---

### Windows

**1. Run the decoder**
The program is standalone — just double-click `RinoDecoder.exe` or run it from a Command Prompt:
```
RinoDecoder.exe
```

If nothing happens or you see a missing runtime error, install the .NET 8 Runtime from https://dotnet.microsoft.com/download/dotnet/8.0 (choose the Runtime, not the SDK), then try again.

**2. Edit settings.json (first time setup)**
Open `settings.json` in any text editor and set your centre frequency, and optionally your latitude/longitude for distance calculations:
```json
"CenterFrequency": "462700000",
"UserLatitude": 0,
"UserLongitude": 0
```

Decoded positions will print to the console. A KML file (`live.kml`) is written to the same folder and updated on each successful decode.
---

### Linux

**1. Run the decoder**
Open a terminal in the RinoDecoder folder and run:
```bash
./RinoDecoder
```

If you see a missing runtime error, install .NET 8 first:
```bash
# Ubuntu/Debian
sudo apt-get update && sudo apt-get install -y dotnet-runtime-8.0

# Fedora/RHEL
sudo dnf install dotnet-runtime-8.0
```

Then run `./RinoDecoder` again.

**2. Edit settings.json (first time setup)**
```bash
nano settings.json
```
Set your centre frequency and coordinates as above.
---

### Viewing decoded positions in Google Earth

Each successful decode appends the position to `live.kml` in the program folder. Google Earth connects via `googleearthloader.kml`, which acts as a Network Link and automatically refreshes the map as new positions come in.

**Google Earth Pro (desktop):**
1. Open Google Earth Pro
2. Go to **File → Open** and select `googleearthloader.kml`
3. Decoded positions appear as placemarks on the map, labelled with the callsign
4. Google Earth will automatically poll `live.kml` and update the map as new positions are decoded — no manual refresh needed

**Google Earth web (earth.google.com):**
1. Click the menu (☰) → **Import KML file**
2. Select `googleearthloader.kml`
3. Positions appear on the map and update automatically as new decodes come in

Each placemark includes the callsign, latitude, longitude, altitude, and the time of decode.

---

## settings.json Reference

### ProgramSettings

Controls the SDR front end and capture pipeline.

| Setting | Default | Description |
|---|---|---|
| `SampleRate` | `250000` | IQ sample rate in samples/sec. Set SDRconnect to 250 ksps. |
| `CenterFrequency` | `462700000` | SDR centre frequency in Hz. 462.7 MHz is the Garmin RINO channel (FRS channel 1 / GMRS). |
| `Modulation` | `NFM` | SDRconnect modulation mode. NFM (narrowband FM) is correct for RINO. |
| `DspDebugLevel` | `1` | Verbosity. `0` = silent, `1` = decoded positions only, `2` = segment table + signal params, `3` = full bit-level debug including eye diagram and M&M tracking. |
| `Threshold` | `0.0010` | Amplitude gate for burst detection in the capture pipeline. IQ magnitude must exceed this value to trigger a capture window. Raise if false triggers occur on a noisy channel; lower if weak signals are being missed. |
| `SaveRawIq` | `true` | If true, saves the raw IQ of the last detected burst to `RawBurstFile` for diagnostic use. |
| `RawBurstFile` | `"raw_burst.cf32"` | Path for the saved raw burst IQ file (complex float32, interleaved I/Q). Used when `SaveRawIq` is enabled. |
| `MaxDecodes` | `4` | Maximum number of burst decode attempts per capture window before giving up. |

---

### DecoderSettings

Controls the signal analyser and decoder. These are auto-calibrated per capture — most values are thresholds that define what counts as a valid preamble or burst.

| Setting | Default | Description |
|---|---|---|
| `PreRollMs` | `0.003` | Seconds of IQ prepended to the detected burst before decoding. Compensates for the burst detector's onset latency. |
| `PostRollMs` | `0.003` | Seconds of IQ appended after the burst end. |
| `MinBurstMs` | `265.0` | Minimum data burst duration in ms. The RINO data burst is ~270 ms; this is the lower acceptance bound. |
| `MaxBurstMs` | `290.0` | Maximum data burst duration in ms. Upper acceptance bound. |
| `MinPreambleMs` | `50.0` | Minimum preamble duration in ms to be considered valid. Short blips below this are ignored. |
| `MaxPreambleMs` | `400.0` | Maximum preamble duration in ms. Longer regions are discarded (likely voice or another signal). |
| `WindowSamplesMs` | `0.005` | Analysis window width in seconds (5 ms). Each window computes mean, standard deviation, and spectral characteristics of the demodulated signal. Smaller = finer time resolution but noisier estimates. |
| `StrideSamplesMs` | `0.001` | Window stride in seconds (1 ms). Controls overlap between consecutive windows. Smaller strides give smoother detection at higher CPU cost. |
| `BaudRatio` | `1.5625` | Ratio of data baud to preamble baud (= 25/16). The measured preamble baud is multiplied by this to derive the data baud rate. |
| `ManualLpfHz` | `0.0` | Override the auto-calculated demod LPF cutoff in Hz. Set to 0 for automatic. Use for debugging only. |
| `LpfBaudMultiplier` | `1.5` | Sets the demod LPF cutoff as a multiple of the data baud rate. Lower reduces noise but can distort symbols; higher passes more noise. |
| `PreambleMaxStd` | `1500` | Maximum signal standard deviation (Hz) for a window to be classified as preamble. The preamble is a clean, stable tone; its std is typically 400–600 Hz. Noise and data bursts exceed this. |
| `PreambleMinStd` | `200` | Minimum signal standard deviation for a preamble window. A silent carrier has std < 100 Hz and must be excluded. |
| `PreambleLowFreqConc` | `0.5` | Minimum low-frequency concentration for a preamble window. Measures how much of the demodulated signal's power sits in low-frequency bins. A slow alternating tone scores high; noise and voice score low. |
| `BurstStdMultiplier` | `2.0` | Burst detection gate. A post-preamble region is treated as a data burst candidate when its signal standard deviation exceeds `preamble_avg_std × BurstStdMultiplier`. Increase if preamble regions are being misidentified as bursts; decrease if weak bursts are being missed. |

---

### RinoSettings

Controls position decoding and output.

| Setting | Default | Description |
|---|---|---|
| `UserLatitude` | `0` | Observer latitude in decimal degrees. Set to your location for distance calculations and improved decoding accuracy. Set to 0 to disable. |
| `UserLongitude` | `0` | Observer longitude in decimal degrees. Set to your location alongside `UserLatitude`. |
| `BitDrift` | `2` | Number of bit positions either side of alignment to try when the frame sync is ambiguous. The decoder tries all 16 polarity/offset combinations; `BitDrift` sets the search width around each candidate. |
| `RetryOnFail` | `true` | If the primary decode fails (bad CRC), retry with inverted polarity and all bit offsets within `BitDrift`. |
