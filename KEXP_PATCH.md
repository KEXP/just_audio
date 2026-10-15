# KEXP patch of just_audio 0.10.6

Local copy of [just_audio 0.10.6](https://pub.dev/packages/just_audio/versions/0.10.6)
(the `example/` folder is omitted), used via `dependency_overrides` in the app's
`pubspec.yaml`.

## Why

KEXP's HLS streams carry the song title/artist as ID3 timed metadata at the
start of each packed-audio (`.aac`) segment (`TIT2`, `TPE1`, `WXXX`). Upstream
just_audio only forwards ICY metadata, so on HLS `icyMetadataStream` never
fires. This patch maps the ID3 title/artist into `IcyInfo` so the app can detect
song changes from the stream without polling.

## Changes

- `android/src/main/java/com/ryanheise/just_audio/AudioPlayer.java`,
  `onMetadata`: handle `TextInformationFrame` (`TIT2`/`TPE1`) and
  `UrlLinkFrame`.
- `darwin/just_audio/Sources/just_audio/AudioPlayer.m`,
  `metadataOutput:didOutputTimedMetadataGroups:fromPlayerItemTrack:`: accept
  `id3/TIT2`, `id3/TPE1`, `id3/WXXX`.

Both produce `title = "Artist - Title"`, matching the ICY `StreamTitle` shape.

HLS also has no ICY headers, so both platforms report the bitrate (bits/s) of
the variant being played as `icyMetadata.headers.bitrate`, updated when
adaptive playback switches variants:

- Android: an `AnalyticsListener.onAudioInputFormatChanged` reads
  `Format.bitrate`.
- iOS: `AVPlayerItemNewAccessLogEntryNotification` reads the access log's
  `indicatedBitrate`.

Both platforms also report the **actual** bitrate (bits/s) in
`icyMetadata.headers.metadataInterval`, a field that's unused on HLS, and set
`headers.name` to `kexp-hls` to mark these headers as synthetic:

- Android: `AnalyticsListener.onLoadCompleted` computes bytes × 8 ÷ media
  duration for each media segment.
- iOS: the access log's `averageAudioBitrate`, re-read on each new access log
  entry and on every timed metadata delivery.

Reusing `metadataInterval` avoids changing `just_audio_platform_interface`;
replace it with a real field before anything user-facing depends on it.

Lines marked `TODO(KEXP)` are temporary verification logging, tagged
`KEXP-HLS` on both platforms:

- Android: ID3 entries, the parsed multivariant and media playlists (raw tags,
  variants, and each new segment's `#EXTINF` title and program date time),
  live offset, track groups, format changes, loads, bandwidth estimates,
  decoder and errors.
- iOS: timed metadata (with the PRIV transport stream timestamp decoded),
  asset metadata, variants, media selection options, `EXT-X-DATERANGE`
  groups, access log entries and the extended access log, error log entries,
  live offset, and track formats and metadata.

## Upgrading just_audio

Copy the new version from the pub cache over this folder (without `example/`),
then re-apply the two changes above.
