---
layout: default
title: OxideAV — Pure-Rust media framework
---

# OxideAV

**Pure-Rust media transcoding and streaming.**

No C libraries. No FFI wrappers. No `*-sys` crates. Every codec,
container, and filter is implemented from the spec in safe Rust, so
you can `cargo add` your way to a working pipeline without any system
dependencies beyond a Rust toolchain.

## Get started

```sh
cargo install oxideav-cli
oxideav list                           # see what's compiled in
oxideav probe input.mp4                # demux probe
oxideav transcode input.flac out.ogg   # quick remux / transcode
```

Or use it as a library:

```toml
[dependencies]
oxideav = "0.0"
```

Full workspace, CLI, and reference player live in
[`oxideav-workspace`](https://github.com/OxideAV/oxideav-workspace).

### Just one codec?

Every codec is its own crate with a minimal footprint — pull only what
you need, without containers, pipelines, or the CLI. The only
infrastructure you need is `oxideav-core` (types) and `oxideav-codec`
(traits + registry):

```toml
[dependencies]
oxideav-core = "0.0"
oxideav-codec = "0.0"
oxideav-g711 = "0.0"     # or any other codec crate
```

```rust
use oxideav_codec::CodecRegistry;
use oxideav_core::{CodecId, CodecParameters, Frame, Packet, TimeBase};

let mut reg = CodecRegistry::new();
oxideav_g711::register(&mut reg);

let mut params = CodecParameters::audio(CodecId::new("pcm_mulaw"));
params.sample_rate = Some(8_000);
params.channels = Some(1);

let mut dec = reg.make_decoder(&params)?;
dec.send_packet(&Packet::new(0, TimeBase::new(1, 8_000), encoded_bytes))?;
let Frame::Audio(a) = dec.receive_frame()? else { unreachable!() };
// a.data[0] is S16 PCM
```

Each codec crate's README has a concrete example tailored to its own
payload shape (see
[`oxideav-g711`](https://github.com/OxideAV/oxideav-g711),
[`oxideav-gsm`](https://github.com/OxideAV/oxideav-gsm),
[`oxideav-mp1`](https://github.com/OxideAV/oxideav-mp1),
[`oxideav-g728`](https://github.com/OxideAV/oxideav-g728),
[`oxideav-gif`](https://github.com/OxideAV/oxideav-gif), …). The
canonical walkthrough of the `send_packet` / `receive_frame` /
`flush` / `reset` loop lives in
[`oxideav-codec`'s README](https://github.com/OxideAV/oxideav-codec).

## Architecture

OxideAV is a modular framework, shipped as ~55 small crates that each
do one thing and publish on their own cadence.

**Infrastructure** — shared types and traits every format crate uses:

- [`oxideav-core`](https://github.com/OxideAV/oxideav-core) — `Packet`, `Frame`, `TimeBase`, `PixelFormat`, `Error`
- [`oxideav-codec`](https://github.com/OxideAV/oxideav-codec) — `Decoder` / `Encoder` traits + registry
- [`oxideav-container`](https://github.com/OxideAV/oxideav-container) — `Demuxer` / `Muxer` traits + probe registry
- [`oxideav-pixfmt`](https://github.com/OxideAV/oxideav-pixfmt) — pixel-format conversion, palette, dither

**Per-format crates** — pick only what you need. 45 separate repos
covering every codec, container, subtitle parser, and source driver in
the ecosystem.

**Binaries** — `oxideav` (CLI: probe / remux / transcode / run) and
`oxideplay` (SDL2 + TUI reference player, SDL2 loaded at runtime via
libloading so no build-time system dep).

## Format coverage

**Containers**: MP4 · Matroska / WebM · Ogg · AVI · FLAC · MP3 · WAV / slin · IFF / 8SVX · PNG / APNG · GIF · JPEG · WebP · AMV · MOD · S3M

**Audio**: PCM · AAC-LC · FLAC · Vorbis · Opus · CELT · Speex · MP1 / MP2 / MP3 · GSM · G.711 · G.722 · G.723.1 · G.728 · G.729

**Video**: MJPEG · FFV1 · MPEG-1 · MPEG-4 Part 2 · H.263 · H.264 (partial) · H.265 (parse) · AV1 (parse) · VP8 · VP9 (partial) · Theora · ProRes 422 · AMV

**Image**: PNG / APNG · GIF · WebP (lossy + lossless) · JPEG

**Subtitles**: SRT · WebVTT · ASS / SSA · TTML · SAMI · MicroDVD · MPL2 · MPsub · VPlayer · PJS · AQTitle · JACOsub · RealText · SubViewer 1/2 · EBU STL · PGS · DVB · VobSub

## Seeking

Container-level seek is implemented for MP4 · MKV / WebM · AVI · FLAC ·
Ogg · MP3 · WAV, using each format's native index (sample-table, Cues,
idx1, SEEKTABLE, page-granule bisection, Xing/VBRI TOC, or byte-offset
arithmetic). Decoders override `Decoder::reset()` to wipe LPC memory /
IMDCT overlap / reference pictures on seek, so the post-seek audio is
clean rather than glitched.

## Contributing

Every crate is its own git repo under
[github.com/OxideAV](https://github.com/OxideAV). CI runs per-crate
(ubuntu + macOS + Windows). Release automation uses
[release-plz](https://release-plz.dev) — every push to `master`
auto-opens a version-bump PR; merging it publishes to crates.io.

Dev workflow for hacking on multiple crates at once: clone
[`oxideav-workspace`](https://github.com/OxideAV/oxideav-workspace),
clone any sibling crate next to it, and run `scripts/dev-patch.sh`.
The generated `.cargo/config.toml` wires the local checkouts via
`[patch.crates-io]`, so `cargo run -p oxideplay` uses your edits
immediately without publishing.

## License

Every crate is MIT-licensed. Copyright © 2026 Karpelès Lab Inc.
