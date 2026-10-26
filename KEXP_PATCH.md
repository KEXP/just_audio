# KEXP patch of just_audio 0.10.6

Fork of [ryanheise/just_audio](https://github.com/ryanheise/just_audio) based on
the [0.10.6](https://pub.dev/packages/just_audio/versions/0.10.6) release. Only
the `just_audio/` package is patched; `just_audio_platform_interface/` and
`just_audio_web/` are unmodified. The KEXP app uses it as a git dependency in
`dependency_overrides` in its `pubspec.yaml`:

```yaml
dependency_overrides:
  just_audio:
    git:
      url: https://github.com/KEXP/just_audio.git
      ref: 0.10.6-kexp.2
      path: just_audio
```

Releases are tagged `<upstream-version>-kexp.<n>` on the `kexp/<upstream-version>`
branch. Never move a tag: for each change, commit to the branch, tag the next
`<n>`, push the tag (`git push origin <tag>`), and update `ref:` in the app.

## Why

KEXP's HLS streams carry the song title/artist as ID3 timed metadata at the
start of each packed-audio (`.aac`) segment (`TIT2`, `TPE1`, `WXXX`). Upstream
just_audio only forwards ICY metadata, so on HLS `icyMetadataStream` never
fires. This patch maps the ID3 title/artist into `IcyInfo` so the app can detect
song changes from the stream without polling.

## Changes

- `just_audio/android/src/main/java/com/ryanheise/just_audio/AudioPlayer.java`,
  `onMetadata`: handle `TextInformationFrame` (`TIT2`/`TPE1`) and
  `UrlLinkFrame`.
- `just_audio/darwin/just_audio/Sources/just_audio/AudioPlayer.m`,
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

`0.10.6-kexp.1` also included temporary `KEXP-HLS` verification logging, which
later releases remove.

## Upgrading just_audio

1. Fetch upstream and create `kexp/<new-version>` from the commit released as
   `<new-version>`.
2. Cherry-pick or rebase the KEXP commits onto it, resolving conflicts in the
   two files above.
3. Push the branch, tag `<new-version>-kexp.1` and push the tag, then set
   `ref:` in the app's `pubspec.yaml` to the tag, run `flutter pub get`, and
   commit the updated `pubspec.lock`.
4. Run `test/unit/services/just_audio_patch_test.dart` in the app. It fails if
   the patch was lost.

