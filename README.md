# AppADay 102 - Reel Splitter

**Split a long MP4 into smaller playable files, or pull the audio out on its own. Nothing is re-encoded and nothing is uploaded.**

**Live:** https://augustineiacopelli.github.io/appaday-102-reel-splitter/

**Category:** Utility (U) &middot; **Shipped:** 2026-08-17

---

## What it does

Load an `.mp4` and Reel Splitter reads its internal index, shows you every keyframe on a film strip, and cuts the file into as many parts as you want. Divide it into equal parts, cap each part at a set number of minutes, or scrub the video and place your own cuts by hand. Every cut lands on a keyframe, so each part opens and plays from its first frame.

It can also throw the picture away and keep only the sound, writing `.m4a` files that are typically a twentieth the size of the source. For a recorded lecture headed somewhere with an upload limit, that alone usually solves the problem without any splitting at all.

## How it works

Most browser video tools either upload your file to a server or load a WebAssembly build of FFmpeg and re-encode everything. Reel Splitter does neither. It parses the MP4 box structure directly in vanilla JavaScript, walks the sample tables, and writes a fresh index for each segment while copying the original frame bytes untouched.

The relevant work happens in four steps:

1. **Find the index.** Top-level boxes are scanned with `File.slice()` until `moov` is located, whether it sits at the front of the file or the back. Only that box is read into memory.
2. **Decode the sample tables.** `stts`, `ctts`, `stss`, `stsc`, `stsz`/`stz2` and `stco`/`co64` are unpacked into parallel typed arrays, giving every sample its byte offset, size, decode time, composition offset, and sync flag. A 40-minute lecture is roughly 175,000 samples and this stays well under a second.
3. **Plan the cut.** Requested times snap to the nearest sync sample, which becomes a segment boundary. Audio samples are selected to match the resulting time span.
4. **Write the segment.** A new `moov` is built with trimmed and re-run-length-encoded tables, chunk offsets are resolved in a two-pass layout, and the output is assembled as a `Blob` whose body is a list of `File.slice()` references into the original. The file's bytes are never held in memory, so a multi-gigabyte source costs almost nothing.

Because the samples are copied rather than decoded, output is bit-for-bit lossless. Decoding all the parts and concatenating the result reproduces the source exactly.

### Two details worth naming

**Composition offsets.** Files with B-frames carry a `ctts` table. Cutting mid-stream leaves the first frame with a non-zero presentation offset, so offsets are normalized by subtracting the segment minimum. Presentation order is preserved and the segment starts at zero.

**AAC needs a running start.** An AAC decoder cannot correctly reconstruct the first frame it sees, because the MDCT overlaps with the frame before it. Every segment after the first therefore carries two extra audio frames ahead of the cut, trimmed back off through an edit list (`elst`). The first segment instead carries the source's own encoder priming value. Without this, each part loses about 23 milliseconds of audio and drifts out of sync with the picture.

## What it handles

- H.264 and HEVC video, AAC audio
- `.mp4`, `.m4v`, `.mov`
- `moov` at the front of the file or at the end
- Files with or without B-frames
- Audio-only sources
- Files larger than 4 GB (64-bit `co64` chunk offsets and `mdat` headers throughout)

WebM and MKV are a different container format and are not supported.

## Verified

Tested against real encodes rather than synthetic box structures.

- Decoded video from a three-way split is **bit-identical** to the source
- Decoded audio from a two-way split is **bit-identical** to the source
- Frame counts reconcile exactly across parts
- A 40-minute, 114 MB file splits twelve ways in about 2.4 seconds under a 512 MB heap cap
- No horizontal overflow at 375px, no console errors, downloads confirmed working
- Unsupported files fail with a clear message and the app recovers on the next load

## Build notes

Single file. Vanilla HTML, CSS and JavaScript, no framework, no build step, no dependency beyond Google Fonts. Nothing is sent anywhere and nothing is stored; the file exists only in the tab while it is open.

Big Shoulders Display for the headings, Archivo for text, Space Mono for timecodes and data. The film strip is the signature element: sprocket perforations in CSS, keyframe ticks drawn on canvas, splice marks in red.

---

Part of [AppADay](https://augustineiacopelli.github.io/appaday/), one complete web app shipped every day by Augustine Iacopelli.
