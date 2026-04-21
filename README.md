# .tapkokit File Format Specification

Version: 1.0

## Overview

A `.tapkokit` file is a standard **ZIP archive** (using either Store or Deflate compression) containing a drum kit definition and its associated audio samples. The file extension is `.tapkokit` but the binary format is a valid ZIP file that can be opened with any ZIP-compatible tool.

## Archive Structure

### Standard Layout (Recommended)

```
MyKit.tapkokit (ZIP)
├── kit.json            ← Kit definition (required)
└── Samples/            ← Sample audio files
    ├── kick.wav
    ├── snare.wav
    ├── hihat_closed.wav
    └── ...
```

### Flat Layout (Also Accepted)

```
MyKit.tapkokit (ZIP)
├── kit.json            ← Kit definition (required)
├── kick.wav            ← Samples alongside kit.json
├── snare.wav
└── ...
```

### Wrapped Layout (Also Accepted)

Some ZIP tools add a root directory wrapper. The importer searches one level deep for `kit.json`:

```
MyKit.tapkokit (ZIP)
└── MyKit/
    ├── kit.json
    └── Samples/
        ├── kick.wav
        └── ...
```

## kit.json Schema

The `kit.json` file defines the kit metadata and all 16 pads.

### Top-Level Object

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `name` | string | Yes | — | Display name of the kit |
| `version` | integer | Yes | — | Schema version. Must be `1`. |
| `createdAt` | string | Yes | — | ISO 8601 date (e.g., `"2026-03-25T00:00:00Z"`) |
| `pads` | array | Yes | — | Array of exactly 16 pad objects |
| `reverbRoomSize` | float | No | `0.5` | Global reverb room size: 0.0–1.0. Controls delay line lengths. |
| `reverbDecay` | float | No | `0.5` | Global reverb decay: 0.0–1.0. Controls feedback and damping. |
| `mainVolume` | float | No | `1.0` | Global kit output volume: 0.0–2.0 (1.0 = unity). |

### Pad Object

All fields marked "No" in the Required column can be safely omitted — the importer uses `decodeIfPresent` with the listed default for every optional field. This means kit JSON files from older versions (or third-party tools) that don't include newer fields like `reverbSend`, `fadeOut`, `sampleMode`, `outputBus`, etc. will import correctly with sensible defaults.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `index` | integer | Yes | — | Pad index, 0–15 |
| `name` | string | Yes | — | Display name (e.g., "Kick", "Snare") |
| `midiNote` | integer | Yes | — | MIDI note number, 0–127 |
| `chokeGroup` | integer | Yes | — | Choke group: 0 = none, 1–8 = choke group ID |
| `volume` | float | Yes | — | Volume multiplier: 0.0–2.0 (1.0 = unity) |
| `sampleFilename` | string or null | Yes | — | Filename of the sample in `Samples/`. `null` = empty pad. |
| `eqLow` | float | No | `0` | Low shelf EQ gain in dB (-12 to +12). 200 Hz. |
| `eqMid` | float | No | `0` | Mid peak EQ gain in dB (-12 to +12). 1 kHz. |
| `eqHigh` | float | No | `0` | High shelf EQ gain in dB (-12 to +12). 5 kHz. |
| `pan` | float | No | `0` | Stereo pan: -1.0 (left) to +1.0 (right). 0.0 = center. |
| `reverbSend` | float | No | `0` | Reverb send level: 0.0 (dry) to 1.0 (full send). |
| `outputBus` | integer | No | `0` | Output bus assignment: 0 = Main, 1–8 = Aux bus. For multi-output hosts. |
| `trimStart` | integer or null | No | `null` | Trim start frame offset (single-sample mode). `null` = beginning. |
| `trimEnd` | integer or null | No | `null` | Trim end frame offset (single-sample mode). `null` = full length. |
| `fadeOut` | float | No | `2.0` | Envelope release/fade out time in milliseconds. Range: 1–2000. |
| `sampleMode` | string | No | `"single"` | `"single"`, `"velocityLayers"`, or `"roundRobin"`. |
| `layers` | array or null | No | `null` | Velocity layers array. See VelocityLayer below. |
| `roundRobinMode` | string | No | `"sequential"` | `"sequential"` or `"random"`. Only used when sampleMode is `"roundRobin"`. |
| `roundRobinSamples` | array or null | No | `null` | Round robin samples array. See RoundRobinSample below. |

### VelocityLayer Object (Optional)

When the `layers` array is present and non-empty, the pad uses velocity-sensitive sample selection. Each layer maps a MIDI velocity range to a different sample.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `sampleFilename` | string or null | Yes | Sample filename. `null` = empty slot. |
| `velocityLow` | integer | Yes | Low end of velocity range, 1–127 inclusive |
| `velocityHigh` | integer | Yes | High end of velocity range, 1–127 inclusive |
| `trimStart` | integer or null | No | Trim start frame offset for this layer. `null` = beginning. |
| `trimEnd` | integer or null | No | Trim end frame offset for this layer. `null` = full length. |
| `fadeOut` | float or null | No | Per-layer fade out in ms. `null` = use pad's `fadeOut` value. |

**Rules for velocity layers:**
- Maximum 5 layers per pad
- Velocity ranges should be contiguous and cover 1–127 with no gaps
- `velocityLow` must be ≤ `velocityHigh`
- When `layers` is absent or `null`, the pad uses `sampleFilename` as a single sample for all velocities

### RoundRobinSample Object (Optional)

When `sampleMode` is `"roundRobin"` and `roundRobinSamples` is present, the pad cycles through multiple samples on successive hits.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `filename` | string | Yes | Sample filename in `Samples/`. |
| `trimStart` | integer or null | No | Trim start frame offset. `null` = beginning. |
| `trimEnd` | integer or null | No | Trim end frame offset. `null` = full length. |
| `fadeOut` | float or null | No | Per-sample fade out in ms. `null` = use pad's `fadeOut` value. |

**Rules for round robin:**
- Maximum 5 samples per pad
- Empty slots are skipped during cycling
- `roundRobinMode` controls cycling: `"sequential"` (1→2→3→1...) or `"random"`
- Round robin position resets on kit switch

### Example kit.json

```json
{
  "name": "My Drum Kit",
  "version": 1,
  "createdAt": "2026-03-25T12:00:00Z",
  "reverbRoomSize": 0.4,
  "reverbDecay": 0.3,
  "mainVolume": 1.0,
  "pads": [
    {
      "index": 0,
      "name": "Kick",
      "midiNote": 36,
      "chokeGroup": 0,
      "volume": 1.0,
      "sampleFilename": "kick.wav",
      "eqLow": 2.0,
      "eqMid": 0,
      "eqHigh": 0,
      "pan": 0,
      "reverbSend": 0.1,
      "outputBus": 1,
      "trimStart": null,
      "trimEnd": null,
      "fadeOut": 5.0,
      "sampleMode": "single"
    },
    {
      "index": 1,
      "name": "Snare",
      "midiNote": 38,
      "chokeGroup": 0,
      "volume": 1.0,
      "sampleFilename": "snare_med.wav",
      "reverbSend": 0.3,
      "outputBus": 2,
      "fadeOut": 10.0,
      "sampleMode": "velocityLayers",
      "layers": [
        { "sampleFilename": "snare_soft.wav", "velocityLow": 1, "velocityHigh": 42, "trimStart": null, "trimEnd": null },
        { "sampleFilename": "snare_med.wav", "velocityLow": 43, "velocityHigh": 84, "trimStart": 1024, "trimEnd": 22050 },
        { "sampleFilename": "snare_hard.wav", "velocityLow": 85, "velocityHigh": 127, "trimStart": null, "trimEnd": null }
      ]
    },
    {
      "index": 2,
      "name": "Closed HH",
      "midiNote": 42,
      "chokeGroup": 1,
      "volume": 1.0,
      "sampleFilename": "hihat_closed.wav",
      "sampleMode": "roundRobin",
      "roundRobinMode": "sequential",
      "roundRobinSamples": [
        { "filename": "hihat_closed_1.wav", "trimStart": null, "trimEnd": null },
        { "filename": "hihat_closed_2.wav", "trimStart": null, "trimEnd": null },
        { "filename": "hihat_closed_3.wav", "trimStart": 100, "trimEnd": 20000 }
      ]
    },
    {
      "index": 3,
      "name": "Open HH",
      "midiNote": 46,
      "chokeGroup": 1,
      "volume": 0.85,
      "sampleFilename": "hihat_open.wav",
      "reverbSend": 0.2,
      "fadeOut": 50.0
    }
  ]
}
```

*Note: The example above shows only 4 pads for brevity. A valid kit.json must contain exactly 16 pad objects (index 0–15). Empty pads should have `"sampleFilename": null`. All optional fields can be omitted — they will use their default values.*

### Minimal Empty Pad

```json
{
  "index": 4,
  "name": "Empty",
  "midiNote": 41,
  "chokeGroup": 0,
  "volume": 1.0,
  "sampleFilename": null
}
```

## Audio File Requirements

### Supported Formats

| Extension | Format |
|-----------|--------|
| `.wav` | WAV (PCM recommended) |
| `.aif` / `.aiff` | AIFF |
| `.mp3` | MPEG Layer 3 |
| `.m4a` | AAC / Apple Lossless |
| `.caf` | Core Audio Format |
| `.flac` | Free Lossless Audio Codec |

### Recommendations

- **Preferred format:** 16-bit or 24-bit WAV at 44100 Hz
- **Sample rate:** Any sample rate is accepted; Tapko converts to the host's sample rate on load
- **Channels:** Mono or stereo. Mono files are played as dual-mono.
- **Duration:** One-shot samples (no looping). Typical drum samples are under 2 seconds.
- **File size:** Keep individual samples under 10 MB. Total kit archive should be under 50 MB for reasonable download times.

### Filename Rules

- Filenames are case-sensitive
- Use only ASCII letters, digits, hyphens, underscores, and dots
- Avoid spaces (use underscores instead)
- Filenames in `kit.json` must exactly match filenames in the archive
- No path traversal (`../`) or absolute paths (`/`) allowed

## ZIP Archive Requirements

### Compression Methods

| Method | Supported |
|--------|-----------|
| Store (method 0) | Yes — recommended for export |
| Deflate (method 8) | Yes — accepted on import |
| Other methods | Not supported — will fail with "Not a valid .tapkokit file" |

### Security Constraints

The importer enforces these safety checks:
- **Path traversal protection:** Filenames containing `../`, starting with `/`, or starting with `..` are rejected
- **Symlink resolution:** The resolved file path must stay within the extraction directory
- **Decompression size limit:** Individual files larger than 256 MB after decompression are rejected
- **ZIP magic number:** The file must start with `PK\x03\x04` (standard ZIP local file header signature)

## Default MIDI Note Mapping

Tapko uses a General MIDI drum layout by default:

| Pad Index | MIDI Note | Default Name |
|-----------|-----------|-------------|
| 0 | 36 | Kick |
| 1 | 38 | Snare |
| 2 | 42 | Closed HH |
| 3 | 46 | Open HH |
| 4 | 41 | Low Floor Tom |
| 5 | 43 | Hi Floor Tom |
| 6 | 45 | Low Tom |
| 7 | 47 | Low-Mid Tom |
| 8 | 48 | Hi-Mid Tom |
| 9 | 50 | High Tom |
| 10 | 49 | Crash 1 |
| 11 | 51 | Ride 1 |
| 12 | 39 | Hand Clap |
| 13 | 54 | Tambourine |
| 14 | 56 | Cowbell |
| 15 | 37 | Side Stick |

Kit authors can override MIDI notes per pad. The mapping above is just the default used by `Default Kit`.

## Server API Integration

If you're building a kit distribution server, the expected API contract is:

### List Kits

```
GET /api/kits
```

Response:
```json
{
  "kits": [
    {
      "id": "my_drum_kit",
      "name": "My Drum Kit",
      "description": "A punchy acoustic kit",
      "author": "Your Name",
      "sampleCount": 8,
      "fileSize": 2048000,
      "downloadURL": "/api/kits/my_drum_kit/download"
    }
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique identifier (URL-safe slug) |
| `name` | string | Display name |
| `description` | string | Short description |
| `author` | string | Kit author name |
| `sampleCount` | integer | Number of samples in the kit |
| `fileSize` | integer | Size of the .tapkokit file in bytes |
| `downloadURL` | string | Relative or absolute URL for downloading |

### Download Kit

```
GET /api/kits/{id}/download
```

Response: The raw `.tapkokit` file (ZIP binary data) with appropriate headers:

```
Content-Type: application/zip
Content-Disposition: attachment; filename="my_drum_kit.tapkokit"
```

**Important:** The response body must be a valid ZIP file. If the server returns HTML, JSON error pages, or other non-ZIP content, the client will report "Not a valid .tapkokit file".

## Validation Checklist

When creating `.tapkokit` files for distribution:

- [ ] File is a valid ZIP archive (starts with `PK\x03\x04`)
- [ ] Contains `kit.json` at the root (or one level deep)
- [ ] `kit.json` has `"version": 1`
- [ ] `kit.json` has exactly 16 pad entries (index 0–15)
- [ ] All `sampleFilename` values reference files that exist in the archive
- [ ] All velocity layer `sampleFilename` values reference files that exist in the archive
- [ ] All round robin `filename` values reference files that exist in the archive
- [ ] Velocity ranges are contiguous (1–127, no gaps) when layers are present
- [ ] `sampleMode` is one of `"single"`, `"velocityLayers"`, or `"roundRobin"` (if present)
- [ ] `outputBus` is 0–8 (if present)
- [ ] Audio files are in a supported format (wav, aif, aiff, mp3, m4a, caf, flac)
- [ ] No path traversal in filenames (no `../` or leading `/`)
- [ ] `createdAt` is a valid ISO 8601 date string
