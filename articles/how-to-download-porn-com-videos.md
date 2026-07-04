# How to Download Porn.com Videos | Technical Analysis of Stream Patterns, CDNs, and Download Methods

This research document provides a practical technical analysis of Porn.com video detection and download workflows. The manifest classifies this target as `generic_embed_or_manifest`, so a reliable implementation should discover media at runtime instead of assuming a fixed CDN, fixed player API, or dedicated extractor. The useful evidence is the rendered page: iframe sources, HTML5 `<video>` and `<source>` elements, `currentSrc`, JW Player or Video.js configuration, KVS-style `kt_player` flashvars, HLS `.m3u8`, DASH `.mpd`, direct MP4/WebM URLs, HAR entries, and the headers and cookies that made the stream accessible.

But first...

## Porn.com Video Downloader Chrome Extension

If you prefer a simpler one-click method, check out the Porn.com Video Downloader browser extension.

Porn.com Downloader is a browser extension built for users who want a cleaner way to save accessible Porn.com videos without falling back to developer tools, network tabs, or command-line utilities. It runs in the active browser session, detects supported media sources from the page and network activity, preserves request context when cookies, referers, origins, or signed URLs are required, and exports the final result as a standard local media file when the stream is supported.

- Save supported Porn.com videos from pages you are authorized to access
- Detect iframe, HTML5 video, JW Player, Video.js, KVS, HLS, DASH, and direct media signals
- Preserve browser-session context where cookies, referers, origins, or signed URLs are required
- Report unsupported DRM or authorization failures clearly instead of pretending they are downloadable
- Use a browser-native workflow instead of separate desktop tools

👉 [Click here to try it free](https://serp.ly/porn-com-downloader?via=github&utm_source=github&utm_medium=piggyback&utm_campaign=howtodownloadvideosagent)

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Porn.com Video Infrastructure Overview](#2-porncom-video-infrastructure-overview)
3. [Embed URL Patterns and Detection](#3-embed-url-patterns-and-detection)
4. [Stream Formats and CDN Analysis](#4-stream-formats-and-cdn-analysis)
5. [yt-dlp Implementation Strategies](#5-yt-dlp-implementation-strategies)
6. [FFmpeg Processing Techniques](#6-ffmpeg-processing-techniques)
7. [Alternative Tools and Backup Methods](#7-alternative-tools-and-backup-methods)
8. [Implementation Recommendations](#8-implementation-recommendations)
9. [Troubleshooting and Edge Cases](#9-troubleshooting-and-edge-cases)
10. [Sources](#10-sources)

---

## 1. Introduction

Porn.com should be handled as a generic embedded-player or manifest-discovery target. The local manifest row identifies the normalized host as `porn.com` and the extractor classification as `generic_embed_or_manifest`. A local exact-name scan of `yt-dlp --list-extractors` during this edit did not return a dedicated entry for this site name or host, so the article avoids site-specific extractor claims and treats the page as a runtime media-detection problem.

This research did not visit explicit media pages and did not download media. It is based on the local manifest, the existing article structure, the local tool versions available in this workspace, and public documentation for `yt-dlp`, FFmpeg, `ffprobe`, `jq`, Chrome DevTools Network/HAR export, MDN HTML media APIs, JW Player, Video.js, HLS, DASH, and Encrypted Media Extensions.

Local tool versions observed during the edit:

```bash
yt-dlp 2025.12.08
ffmpeg 8.0.1
ffprobe 8.0.1
jq-1.7.1
```

The practical goal is not to guess where Porn.com stores videos. The goal is to record enough browser evidence to answer four questions:

1. Is the playable media exposed as a normal URL, a streaming manifest, or an embedded third-party player?
2. Does the URL require the same cookies, referer, origin, user-agent, or query string seen in the browser?
3. Can a standard tool such as `yt-dlp`, `ffmpeg`, or `ffprobe` read the stream without altering those access requirements?
4. Is the stream protected by DRM or an authorization layer that should be reported as unsupported?

The conservative implementation strategy is evidence-first: detect the page host, inspect the rendered DOM, recurse into iframes when permitted, watch network traffic during actual playback, and only then pass a verified page URL or media URL into a downloader.

## 2. Porn.com Video Infrastructure Overview

Known target information from the manifest:

```text
Site name: Porn.com
Domain: porn.com
Output path: content/site-articles/porn-com.md
Extractor classification: generic_embed_or_manifest
```

No fixed CDN hostnames are asserted for Porn.com. For this class of site, hardcoding a CDN list is usually weaker than collecting the real media host from the current browser session. The page may use a first-party player, a cross-origin iframe, a generic HTML5 video element, a player library configuration object, or a manifest URL that appears only after playback begins.

Inspect these surfaces in order:

- Final URL and redirect chain on `porn.com` and `www.porn.com`
- Iframe `src` attributes, sandbox attributes, `allow` permissions, and nested player origins
- HTML5 `<video>` `src`, `currentSrc`, `poster`, and child `<source>` URLs
- Inline JSON, application state, JSON-LD `VideoObject`, OpenGraph video fields, and Twitter player metadata
- JW Player `playlist`, `file`, `sources`, and quality labels
- Video.js `data-setup`, typed `<source>` tags, and programmatic `player.src(...)` calls
- KVS-style `kt_player` setup and `flashvars` values such as `video_url`, `video_alt_url`, and quality-specific URL keys
- Network requests for `.m3u8`, `.mpd`, `.mp4`, `.webm`, `.m4s`, `.ts`, captions, subtitles, thumbnails, key files, and license calls

Treat the browser session as part of the infrastructure. A URL that succeeds in the browser can fail in a command-line tool if the query string is changed, a cookie is missing, the `Referer` is different, or the `Origin` header no longer matches the page. Conversely, a URL that looks unusable in static HTML may become readable once playback starts and the player resolves its manifest.

A useful implementation data model for each candidate is:

```text
source_type: iframe | html5 | jwplayer | videojs | kvs | hls | dash | direct | drm | unknown
page_url: final browser page URL
frame_url: iframe URL when media is inside a nested frame
media_url: manifest or direct media URL when observed
mime_type: response MIME type or source tag type when available
request_headers: referer, origin, user-agent, cookies, range support
evidence: dom | script | har | network | ffprobe | yt-dlp
status: pending | supported | unsupported_drm | unauthorized | expired | failed
```

This keeps the Porn.com implementation explainable: every download attempt should be traceable to a concrete DOM node, player config value, HAR entry, or tool output.

## 3. Embed URL Patterns and Detection

Start with the host rule. It should match the normalized host and the `www` variant, but it should not match unrelated domains that merely contain the same characters.

```regex
https?:\/\/(?:www\.)?porn\.com(?:\/|$)
```

Host examples for this row:

```text
https://porn.com/
https://www.porn.com/
```

A rendered DOM probe should inspect iframes and media elements together. MDN documents `currentSrc` as the selected absolute media URL for an HTML media element, which is why `currentSrc` is more useful than checking only the literal `src` attribute.

```js
Array.from(document.querySelectorAll("iframe, video, source")).map((node, index) => ({
  index,
  tag: node.tagName.toLowerCase(),
  src: node.src || node.currentSrc || node.getAttribute("src") || "",
  currentSrc: node.currentSrc || "",
  type: node.type || node.getAttribute("type") || "",
  sandbox: node.getAttribute("sandbox") || "",
  allow: node.getAttribute("allow") || ""
}));
```

For iframes, record the frame URL and inspect it as a separate page when policy and access allow it. Cross-origin iframes may not expose their internal DOM to the parent page, but their `src` value is still useful. If the iframe host is supported by a known extractor, hand it to that extractor. If it is not supported, use the same generic runtime workflow on the frame URL.

Search static HTML and rendered scripts for media URL patterns, but do not rely on regex alone as the parser. Use regex as a candidate finder and then validate every candidate with headers or a media probe.

```regex
https?:\/\/[^"'\s<>]+?\.(?:m3u8|mpd|mp4|webm|m4s|ts)(?:\?[^"'\s<>]*)?
```

Player configuration search terms that are worth scanning in scripts and rendered HTML:

```text
jwplayer(
playlist
sources
file
videojs(
data-setup
player.src(
kt_player
flashvars
video_url
video_alt_url
mediaDefinition
```

Browser console helpers for player libraries:

```js
// HTML5 media, including the selected currentSrc.
Array.from(document.querySelectorAll("video")).map((video, index) => ({
  index,
  src: video.src,
  currentSrc: video.currentSrc,
  networkState: video.networkState,
  readyState: video.readyState,
  sources: Array.from(video.querySelectorAll("source")).map(source => ({
    src: source.src,
    type: source.type
  }))
}));
```

```js
// Video.js player cache, when the page exposes it.
window.videojs ? Object.values(window.videojs.players || {}).map(player => ({
  id: player.id && player.id(),
  currentSrc: player.currentSrc && player.currentSrc(),
  sources: player.currentSources && player.currentSources()
})) : [];
```

```js
// JW Player instances, when jwplayer is globally available.
window.jwplayer ? Array.from(document.querySelectorAll("[id]")).map(element => {
  try {
    const player = window.jwplayer(element.id);
    const playlist = player && player.getPlaylist ? player.getPlaylist() : null;
    return playlist ? { id: element.id, playlist } : null;
  } catch (error) {
    return null;
  }
}).filter(Boolean) : [];
```

KVS-style players often store media in URL-encoded flashvars or JavaScript configuration. Decode candidates once, inspect keys that look like media URLs, and reject anything that is only a thumbnail, tracking pixel, ad URL, or license endpoint. Do not invent a KVS extractor for Porn.com; treat KVS fields as one possible evidence source.

## 4. Stream Formats and CDN Analysis

Classify observed Porn.com media by evidence, not by expected hostname:

- HLS `.m3u8`: master playlists, media playlists, `#EXTM3U`, `#EXT-X-STREAM-INF`, audio/video renditions, `.ts`, or fragmented MP4 `.m4s` segments
- DASH `.mpd`: MPD manifests with `Representation`, `AdaptationSet`, `BaseURL`, `SegmentTemplate`, or separate audio/video tracks
- Direct MP4/WebM: `<video>` or `<source>` URLs, player `file` values, or direct network responses with `video/mp4` or `video/webm`
- Blob-backed MediaSource: a `blob:` `currentSrc` where the remote manifest or segments appear only in Network logs
- Signed URL: query strings containing values such as `token`, `expires`, `exp`, `policy`, `signature`, `Key-Pair-Id`, `X-Amz-Signature`, or similar access parameters
- DRM/EME: `requestMediaKeySystemAccess`, `MediaKeys`, `widevine`, `fairplay`, `playready`, `pssh`, `ContentProtection`, or license requests

Validate manifest and media candidates before attempting a full download. The commands below use shell variables so the page URL, media URL, and origin remain explicit inputs.

```bash
SITE_ORIGIN='https://porn.com'
read -r PAGE_URL
read -r MEDIA_URL
curl -L --compressed \
  -H "Referer: $PAGE_URL" \
  -H "Origin: $SITE_ORIGIN" \
  "$MEDIA_URL" | sed -n '1,40p'
```

HLS validation checks:

- The first non-empty line should normally be `#EXTM3U`.
- A master playlist usually contains `#EXT-X-STREAM-INF` and variant playlist URLs.
- Media playlists contain segment references and timing tags such as `#EXTINF`.
- Relative segment URLs must be resolved against the playlist URL, not the page URL.
- Encrypted HLS may include `#EXT-X-KEY`; ordinary download tools can handle some clear-key cases but should not bypass DRM.

DASH validation checks:

- The document should contain an `<MPD` root or MPD namespace.
- `AdaptationSet` and `Representation` identify tracks and qualities.
- `SegmentTemplate`, `SegmentList`, `SegmentBase`, or `BaseURL` explains where segment URLs come from.
- Separate audio and video tracks require muxing after retrieval.
- `ContentProtection` or PSSH data is DRM evidence and should trigger unsupported reporting unless the user is using an authorized player workflow.

Direct media validation should use headers and a tiny range request before a full transfer:

```bash
SITE_ORIGIN='https://porn.com'
read -r PAGE_URL
read -r MEDIA_URL
curl -L -sS -D - -o /dev/null -r 0-1 \
  -H "Referer: $PAGE_URL" \
  -H "Origin: $SITE_ORIGIN" \
  "$MEDIA_URL"
```

Useful signs are `200 OK` or `206 Partial Content`, a video MIME type, stable redirects, and a content length that is plausible for media. Warning signs are `401`, `403`, `404`, `410`, very short HTML responses, or redirects back to the page instead of a media object.

## 5. yt-dlp Implementation Strategies

`yt-dlp` is the first command-line tool to try because it can inspect pages, list formats, merge audio and video, pass custom headers, and read cookies from a browser profile. For Porn.com, use the generic extractor path unless a later tool version adds a dedicated extractor.

Generic format listing from the page URL:

```bash
SITE_ORIGIN='https://porn.com'
read -r PAGE_URL
yt-dlp \
  --use-extractors generic \
  --cookies-from-browser chrome \
  --add-headers "Referer:$PAGE_URL" \
  --add-headers "Origin:$SITE_ORIGIN" \
  --list-formats \
  "$PAGE_URL"
```

Metadata dump with a concise `jq` summary:

```bash
read -r PAGE_URL
yt-dlp --use-extractors generic --cookies-from-browser chrome --dump-json "$PAGE_URL" \
  | jq '{
      id,
      title,
      extractor,
      webpage_url,
      live_status,
      format_count: ((.formats // []) | length),
      formats: [(.formats // [])[] | {format_id, ext, protocol, acodec, vcodec, width, height, url: (.url // "")[:120]}]
    }'
```

Print resolved URLs without downloading:

```bash
read -r PAGE_URL
yt-dlp --use-extractors generic --cookies-from-browser chrome --print urls "$PAGE_URL"
```

Download a browser-captured media URL while preserving page context:

```bash
SITE_ORIGIN='https://porn.com'
read -r PAGE_URL
read -r MEDIA_URL
yt-dlp \
  --cookies-from-browser chrome \
  --add-headers "Referer:$PAGE_URL" \
  --add-headers "Origin:$SITE_ORIGIN" \
  --add-headers "User-Agent:Mozilla/5.0" \
  -o '%(extractor)s-%(id)s-%(title).120B.%(ext)s' \
  "$MEDIA_URL"
```

Preferred merged output for page-level extraction:

```bash
SITE_ORIGIN='https://porn.com'
read -r PAGE_URL
yt-dlp \
  --use-extractors generic \
  --cookies-from-browser chrome \
  --add-headers "Referer:$PAGE_URL" \
  --add-headers "Origin:$SITE_ORIGIN" \
  -f 'bv*+ba/b' \
  --merge-output-format mp4 \
  -o '%(extractor)s-%(id)s-%(title).120B.%(ext)s' \
  "$PAGE_URL"
```

If `yt-dlp` reports no formats, that does not prove the page has no media. It means the generic extractor did not resolve media from the page alone. Continue with browser Network capture, iframe inspection, player config extraction, and direct manifest validation.

## 6. FFmpeg Processing Techniques

FFmpeg is useful after the media URL is known. It is less convenient than `yt-dlp` for browser cookies, but it is strong for HLS, DASH, direct media probing, remuxing, and reconnect behavior. FFmpeg HTTP protocol options support custom headers and reconnect settings, which matter for signed or fragile URLs.

Build a reusable header string for `ffmpeg` and `ffprobe`:

```bash
SITE_ORIGIN='https://porn.com'
UA='Mozilla/5.0'
read -r PAGE_URL
read -r MEDIA_URL
HEADERS=$(printf 'Referer: %s\r\nOrigin: %s\r\nUser-Agent: %s\r\n' "$PAGE_URL" "$SITE_ORIGIN" "$UA")
```

Probe stream structure before downloading:

```bash
ffprobe -hide_banner -v error \
  -headers "$HEADERS" \
  -show_entries format=format_name,duration:stream=index,codec_type,codec_name,width,height,avg_frame_rate \
  -of json \
  "$MEDIA_URL" | jq '.'
```

HLS remux to MP4:

```bash
ffmpeg -hide_banner -loglevel warning \
  -headers "$HEADERS" \
  -i "$MEDIA_URL" \
  -map 0 \
  -c copy \
  -bsf:a aac_adtstoasc \
  output.mp4
```

DASH remux to Matroska when tracks or codecs do not fit cleanly into MP4:

```bash
ffmpeg -hide_banner -loglevel warning \
  -headers "$HEADERS" \
  -i "$MEDIA_URL" \
  -map 0 \
  -c copy \
  output.mkv
```

Direct MP4/WebM copy after URL validation:

```bash
ffmpeg -hide_banner -loglevel warning \
  -headers "$HEADERS" \
  -i "$MEDIA_URL" \
  -c copy \
  output.mp4
```

Retry mode for transient HTTP failures:

```bash
ffmpeg -hide_banner -loglevel warning \
  -reconnect 1 \
  -reconnect_streamed 1 \
  -reconnect_on_network_error 1 \
  -reconnect_on_http_error 429,500,502,503,504 \
  -reconnect_delay_max 10 \
  -headers "$HEADERS" \
  -i "$MEDIA_URL" \
  -c copy \
  output.mp4
```

If the URL requires cookies, prefer `yt-dlp --cookies-from-browser` first. If FFmpeg must be used, copy only the minimum necessary cookie context from an authorized browser session and avoid storing sensitive HAR files longer than necessary.

## 7. Alternative Tools and Backup Methods

Browser Network capture is the most important fallback for Porn.com. Chrome DevTools documents that the Network panel records requests made while DevTools is open and can export those requests as a HAR file. Enable capture before playback starts so early manifest requests are not missed.

Browser workflow:

1. Open the authorized Porn.com page in the browser.
2. Open DevTools and select the Network panel.
3. Enable Preserve log so redirects and frame navigations remain visible.
4. Start playback and wait until the player has selected a stream.
5. Filter for `m3u8`, `mpd`, `mp4`, `webm`, `m4s`, `ts`, `vtt`, `key`, `license`, `widevine`, `fairplay`, and `playready`.
6. Export a sanitized HAR by default; export with sensitive data only when you control the session and need cookies for debugging.

`jq` HAR analysis for media and manifest URLs:

```bash
jq -r '
  .log.entries[]
  | {
      status: .response.status,
      method: .request.method,
      url: .request.url,
      mime: (.response.content.mimeType // ""),
      referer: ((.request.headers // [])[]? | select(.name | ascii_downcase == "referer") | .value)
    }
  | select(
      (.url | test("\\.(m3u8|mpd|mp4|webm|m4s|ts|vtt)(\\?|$)"; "i"))
      or (.mime | test("mpegurl|dash|video|mp4|webm|mp2t"; "i"))
    )
  | [.status, .method, .mime, (.referer // ""), .url]
  | @tsv
' browser-session.har
```

`jq` HAR analysis for DRM indicators:

```bash
jq -r '
  .log.entries[]
  | .request.url
  | select(test("license|widevine|fairplay|playready|drm|eme|pssh"; "i"))
' browser-session.har
```

Streamlink can be useful for HLS when the manifest is already known:

```bash
read -r MEDIA_URL
streamlink "$MEDIA_URL" best -o output.ts
```

`aria2c` is appropriate only after resolving a direct file URL, not for an HLS or DASH manifest:

```bash
read -r MEDIA_URL
aria2c --header="Referer: $PAGE_URL" --header="Origin: $SITE_ORIGIN" "$MEDIA_URL"
```

Specialist HLS/DASH tools such as `N_m3u8DL-RE` can help with variant selection, subtitles, and fragmented media, but they should still receive only URLs captured from authorized playback. If DRM evidence appears, route to unsupported reporting rather than trying to defeat encryption.

## 8. Implementation Recommendations

Recommended Porn.com pipeline:

1. Match only `porn.com` and `www.porn.com`; avoid substring matches against unrelated hosts.
2. Resolve redirects and store the final page URL as the canonical referer.
3. Inspect the rendered DOM for iframes, `<video>`, `<source>`, and `currentSrc`.
4. Inspect script state for JW Player, Video.js, and KVS `kt_player` or `flashvars` media keys.
5. Start playback before declaring that no media exists; many players resolve manifests lazily.
6. Capture Network requests and, when needed, export a HAR for deterministic analysis with `jq`.
7. Score candidates by evidence strength: active `currentSrc`, successful media response, valid HLS/DASH manifest, direct video MIME type, then weaker script strings.
8. Try `yt-dlp --use-extractors generic` on the page URL and then on the captured media URL.
9. Use `ffprobe` to validate codecs, tracks, and container shape before a full FFmpeg run.
10. Preserve signed URLs exactly, including query order, escaping, and case-sensitive parameter names.
11. Preserve `Referer`, `Origin`, browser cookies, and a realistic user-agent when the browser used them.
12. Report `unsupported_drm` when EME, license requests, encrypted manifests, or protected initialization data are the only available path.

A small scoring model helps prevent false positives:

```text
+5 active video.currentSrc that is http(s)
+5 HAR response with video MIME type or HLS/DASH MIME type
+4 manifest body validates as HLS or DASH
+3 JW Player, Video.js, or KVS config contains a URL that validates
+2 iframe URL routes to a known provider or generic extractor
-4 URL is a thumbnail, ad, analytics request, or tracking pixel
-5 response is HTML, login page, or blocked status
-8 DRM/license evidence without a clear unencrypted media URL
```

Store enough debug detail to explain failures without exposing secrets. For example, record status code, MIME type, candidate source, and whether cookies or referer were required, but redact full cookies and long signed query strings in logs.

## 9. Troubleshooting and Edge Cases

If the only visible URL is `blob:`, do not pass the blob URL to `yt-dlp` or FFmpeg. A blob URL is a browser object URL, not the remote media. Capture the manifest or media segments from Network while playback is active.

If a captured URL returns `403`, retry with the same `Referer`, `Origin`, user-agent, and cookies used by the browser. If it still fails, the URL may be expired, IP-bound, session-bound, or intentionally blocked outside the player.

If a signed URL contains query parameters, preserve the whole URL exactly. Do not sort parameters, decode and re-encode the query string, remove apparently redundant fields, or change capitalization. Many signing schemes cover the exact path and query bytes.

If the page uses a cross-origin iframe, inspect the iframe URL separately. The parent Porn.com page may contain only an embed shell while the media URL is resolved inside the child frame.

If HLS playback fails in FFmpeg but the manifest validates, check whether the playlist contains relative segment paths, encrypted key references, discontinuities, or a live sliding window. Try `yt-dlp` and `streamlink` before assuming the URL is unusable.

If DASH output has separate audio and video, use a merged format in `yt-dlp` or an FFmpeg remux command that maps all streams. MP4 is convenient, but Matroska can be a safer temporary output when codecs or subtitle tracks do not fit MP4 cleanly.

If `ffprobe` reports no duration, that can be normal for live HLS, short sliding windows, or some fragmented streams. Validate stream presence, codecs, and segment access rather than relying only on duration.

If license requests, `requestMediaKeySystemAccess`, `MediaKeys`, Widevine, FairPlay, PlayReady, PSSH, or manifest `ContentProtection` are present and no unencrypted media URL is available, mark the result as unsupported DRM. The implementation should not attempt DRM bypass.

If no media appears in the first Network capture, reload with Preserve log already enabled, clear filters, start playback again, and watch both the top page and child frame requests. Some players do not request the manifest until a user gesture, quality change, or seek event.

## 10. Sources

- [yt-dlp README and options](https://github.com/yt-dlp/yt-dlp)
- [yt-dlp supported sites list](https://github.com/yt-dlp/yt-dlp/blob/master/supportedsites.md)
- [yt-dlp generic extractor source](https://github.com/yt-dlp/yt-dlp/blob/master/yt_dlp/extractor/generic.py)
- [FFmpeg documentation](https://ffmpeg.org/ffmpeg.html)
- [FFmpeg HTTP protocol options](https://ffmpeg.org/ffmpeg-protocols.html)
- [ffprobe documentation](https://www.ffmpeg.org/ffprobe-all.html)
- [jq manual](https://jqlang.org/manual/)
- [Chrome DevTools Network reference](https://developer.chrome.com/docs/devtools/network/reference/)
- [MDN: HTML video element](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/video)
- [MDN: HTML source element](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/source)
- [MDN: HTML iframe element](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/iframe)
- [MDN: HTMLMediaElement currentSrc](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/currentSrc)
- [MDN: Encrypted Media Extensions API](https://developer.mozilla.org/en-US/docs/Web/API/Encrypted_Media_Extensions_API)
- [Video.js setup guide](https://legacy.videojs.org/guides/setup/)
- [JW Player playlist and sources reference](https://docs.jwplayer.com/players/reference/playlists)
- [RFC 8216: HTTP Live Streaming](https://www.rfc-editor.org/rfc/rfc8216)
- [MPEG-DASH overview from MPEG](https://www.mpeg.org/standards/MPEG-DASH/1/)
