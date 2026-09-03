# Protocol notes: endpoint switches and the type_mask high byte

Verified on hardware: HQPlayer Desktop 5.35 sending to an endpoint running
`networkaudiod` 6.1.4 (NAA protocol v6), 2026-09-02. This supplements the
frame layout already implemented in `src/frame.rs` — it doesn't change it.

## The in-band sections are opt-in on the endpoint, not something HQPlayer decides alone

`networkaudiod` reads three environment variables:

```
NETWORKAUDIOD_METADATA="1"
NETWORKAUDIOD_POSITION="1"
NETWORKAUDIOD_PICTURE="1"
```

Signalyst ships them in `/etc/default/networkaudiod`, commented out. With
them unset, we captured a full track change and a stop/start of a new album
over the NAA TCP connection: every frame had `type_mask=1` (PCM only), and
the start negotiation didn't name position, metadata, or picture at all.
With the three variables set to `1`, the same HQPlayer instance sent the
sections in the same session, unprompted, on the first frame after start.

For anyone troubleshooting a proxy that injects metadata into this stream:
if the endpoint shows nothing, the proxy may be working correctly and the
endpoint's `networkaudiod` may simply have these switches off. That's worth
checking before assuming the proxy or HQPlayer is at fault — a stock
endpoint with the switches unset will show nothing no matter what a proxy
puts on the wire, because the endpoint isn't the thing deciding to render
that section; nothing sent it in the first place until we turned these on.

We also confirmed, independently of any proxy, that with the switches on
the endpoint's daemon writes what it received to disk:

```
<device name>.metadata
<device name>.position
<device name>.picture.txt|jpg|png|gif
```

into `/run/networkaudiod`, written atomically (`.new` then rename).
`.position` is rewritten about once a second; `.metadata` on track change.
For streaming sources (Qobuz, TIDAL in our capture) the picture arrives as
a URL inside `.picture.txt` rather than image bytes — we didn't observe the
local-file case where an embedded image lands as `.jpg`/`.png`/`.gif`
instead.

## type_mask high byte on a `stream="dop"` start

The start negotiation we captured for a DoP stream:

```
HQPlayer -> endpoint: <operation bits="32" channels="2" netbuftime="1" rate="352800" stream="dop" type="start"/>
endpoint -> HQPlayer: <operation bits="32" channels="2" dsd="0" netbuftime="1" rate="352800" result="1" stream="dop" type="start"/>
```

The first frame after that start carried `type_mask=0x0100001d`: the low
byte is `0x1d` (PCM + position + metadata + picture, matching the bits this
project already defines), but the high byte is `0x01`, set. We did not see
that high byte set on PCM or DSD-native streams in our captures — only on
this DoP start's first frame. Subsequent frames in the same stream were
`type_mask=0x00000001` (PCM only, no high byte).

We don't know what the high byte means. It's worth documenting because a
parser that treats anything above `0x1f` as corrupt will drop this frame
outright. If your `is_corrupt` (or
equivalent) validates `type_mask` against a small mask, a DoP stream may hit
this.

## Section order, confirmed against a live capture

Matches this project's documented layout. After the PCM payload, sections
appear in this order when present, each ending in the `\0` this project's
`build_meta_section`/`build_pos_section` also emit:

```
[position]...\0
[metadata]...\0
<picture bytes, or a URL followed by>\0
```

Offsets in our capture landed exactly where `pos_len`/`meta_len`/`pic_len`
predicted (position at `start + 32 + pcm_len`, metadata `pos_len` bytes
after that, picture `meta_len` bytes after that) — no padding or alignment
between sections.

## Source

`networkaudiod` 6.1.4-71, NAA protocol v6, driven by
HQPlayer Desktop 5.35 on macOS. Captured with tcpdump on the machine
running HQPlayer and decoded by hand against this project's frame layout.
