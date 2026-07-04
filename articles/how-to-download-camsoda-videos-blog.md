# How to Download CamSoda Videos | Technical Analysis of Live Authorization, Stream Patterns, CDNs, and Download Methods

This research document provides a technical analysis of CamSoda live video delivery, including room URL patterns, tokenized HLS manifests, live/offline states, private-room handling, and authorized-session download methods. CamSoda is a live-cam platform, so a downloader must treat availability as temporary: a room can be public, offline, private, geo-restricted, or otherwise inaccessible at the moment of extraction.

But first...

## CamSoda Video Downloader Chrome Extension

If you prefer a simpler one-click method, check out the CamSoda Video Downloader browser extension.

CamSoda Downloader is a browser extension built for users who want a cleaner way to save accessible CamSoda videos without falling back to developer tools, network tabs, or command-line utilities. It detects supported media sources in the active browser session and exports the final result as a usable local file for offline playback.

- Save supported CamSoda videos from pages you are authorized to access
- Detect available media sources without manual network inspection
- Preserve browser-session context where cookies, referers, or signed URLs are required
- Export detected video as a standard local media file when supported
- Use a browser-native workflow instead of separate desktop tools

👉 Click here to try it free: https://serp.ly/camsoda-downloader?via=github&utm_source=github&utm_medium=piggyback&utm_campaign=howtodownloadvideosagent

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [CamSoda Video Infrastructure Overview](#2-camsoda-video-infrastructure-overview)
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

CamSoda extraction is primarily a live-stream problem. The installed `yt-dlp` package includes a dedicated `Camsoda` extractor, and the extractor source shows a flow based on the room username, a video-token API call, public/private/offline checks, and HLS manifest extraction.

This research did not visit explicit media pages and did not download site media. Local tool checks showed:

```bash
yt-dlp 2025.12.08
ffmpeg version 8.0.1
```

Important boundary: this workflow must not imply access to private shows, paid entitlements, hidden rooms, or any stream the viewer cannot already access. The extractor itself treats private shows and offline rooms as expected failure states, which is the correct product behavior.

### Research Scope

This document covers:

- CamSoda room URL detection
- Live HLS manifest handling
- Tokenized stream URLs from the official `yt-dlp` extractor logic
- `yt-dlp` and `ffmpeg` commands for public/authorized live streams
- Troubleshooting for private, offline, expired-token, and geo-restricted states

## 2. CamSoda Video Infrastructure Overview

The `yt-dlp` CamSoda extractor matches room pages on `www.camsoda.com` where the path is the room username.

The extractor then calls a video-token endpoint under `/api/v1/video/vtoken/` with the same room username.

The API response is expected to provide the stream metadata needed for HLS extraction. The extractor checks for:

- Empty or missing configuration
- `private_servers`, which indicates a private show
- Missing `stream_name`, which indicates the model is offline
- `edge_servers`, which are used to build HLS URLs
- A `token`, appended to the HLS manifest query string

The generated HLS URL is an HTTPS edge-server URL ending in `_v1/index.m3u8` with a `token` query parameter. Because the edge server comes from the API response, implementations should not hard-code or invent CDN hostnames. The correct host is the one returned for the current session and room state.

### Live State Model

A CamSoda downloader should classify extraction into explicit states:

```text
public_live
offline
private_show
no_stream_configuration
no_active_hls_formats
geo_restricted
expired_token
network_error
```

### Authorization Boundary

The workflow is for streams the user is allowed to view. It must not try to enter private shows, bypass paid interactions, use another user's token, or keep refreshing an endpoint after access has been revoked.


### Browser Session and Authorization Model

For CamSoda, the reliable implementation boundary is the same boundary the browser enforces. The downloader should begin from a page the user can already play, then carry forward only the request context needed for that current session. That context usually includes the page URL, the request URL for the manifest or direct media object, the browser user-agent, cookies scoped to the site, and the `Referer` and `Origin` headers observed on the media request. Do not hard-code CDN hosts or construct media URLs from usernames, room names, post IDs, or visible page slugs.

A conservative capture record should include only the active CamSoda page URL, the final authorized manifest or direct-media URL, the observed `Referer`, `Origin`, `User-Agent`, and cookie context, the HTTP status and content type, the detected transport, and the capture time for the current diagnostic session. The expected origin for first-party browser requests is `https://camsoda.com` unless the HAR proves a different first-party origin for the active page.

The capture should expire with the browser session. Signed query strings, CDN cookies, and entitlement-bearing headers may be valid only for a short time. A useful extension does not try alternate private URLs after a failure; it refreshes the page, re-detects the active media request, and reports the exact authorization or transport state.

Implementation detail that is worth logging:

- Whether playback was actually started before detection
- Whether the room, post, purchase, or clip is public, authenticated, paid, private, offline, or geo-blocked
- Whether the media request returned `200`, `206`, `302`, `401`, `403`, `404`, `410`, or `451`
- Whether the response content type looked like HLS, DASH, MP4/WebM, JavaScript configuration, JSON API metadata, or a license/key request
- Whether the final media URL contained an expiry, signature, policy, token, or CDN cookie dependency
- Whether browser APIs indicated Media Source Extensions, Encrypted Media Extensions, or WebRTC instead of a plain downloadable URL

## 3. Embed URL Patterns and Detection

### Primary Room URL Pattern

Room pages use `www.camsoda.com` followed by the room username.

Room username regex:

```regex
https?:\/\/www\.camsoda\.com\/([\w-]+)(?:[/?#]|$)
```

### API and HLS Evidence

Safe detection should focus on the room page and the video-token response:

```text
/api/v1/video/vtoken/
stream_name
edge_servers
private_servers
token
index.m3u8
```

Browser-side probe:

```js
performance.getEntriesByType("resource")
  .map(entry => entry.name)
  .filter(url =>
    /camsoda\.com\/api\/v1\/video\/vtoken/i.test(url) ||
    /\.m3u8(\?|$)/i.test(url) ||
    /token|stream_name|edge_servers/i.test(url)
  );
```

### HLS Manifest Detection

Regex for observed HLS URLs:

```regex
https?:\/\/[^"'\s<>]+\/[^"'\s<>]+_v1\/index\.m3u8\?[^"'\s<>]*token=[^"'\s<>]+
```

Preserve the full query string. Tokens can be short-lived and tied to the room state.


### Browser, HAR, and Request Inspection Workflow

Use DevTools only after the CamSoda player has begun playback. Chrome DevTools records network requests while the Network panel is open, and the Preserve log setting keeps requests visible across navigation. For live/auth sites, that matters because the manifest request may appear after a room-state API call, a player bootstrap request, or a redirect.

Practical browser checks:

1. Open the CamSoda page that already plays in the browser.
2. Open DevTools, switch to Network, enable Preserve log, and clear the old request list.
3. Start playback and filter by Media, Fetch/XHR, `url:m3u8`, `url:mpd`, `status-code:403`, and likely terms such as `playlist`, `manifest`, `master`, `media`, `segment`, `m4s`, or `ts`.
4. Inspect the selected request Headers tab before copying anything. Confirm the request method, status code, content type, request headers, response headers, and final URL after redirects.
5. Export a HAR only for your own diagnostic session and treat it as sensitive because cookies and signed URLs may be present.

Browser console helpers can identify whether a standard media element has a visible source. They are not bypass methods; they only show what the active page exposes to the browser:

```javascript
Array.from(document.querySelectorAll('video')).map((video) => ({
  currentSrc: video.currentSrc,
  src: video.src,
  readyState: video.readyState,
  networkState: video.networkState,
  duration: video.duration,
}));

performance.getEntriesByType('resource')
  .map((entry) => entry.name)
  .filter((url) => /m3u8|mpd|m4s|\.ts(\?|$)|playlist|manifest/i.test(url));
```

Once a HAR is exported, `jq` can classify candidate requests without opening the media page again:

```bash
har_file="$PWD/camsoda-com-session.har"

jq -r '
  .log.entries[]
  | select(.request.url | test("\\.(m3u8|mpd)(\\?|$)|manifest|playlist|master"; "i"))
  | [.response.status, .response.content.mimeType, .request.url]
  | @tsv
' "$har_file"

jq -r '
  .log.entries[]
  | select(.request.url | test("\\.(m4s|ts|mp4|webm)(\\?|$)"; "i"))
  | [.response.status, .response.headers[]? | select(.name|ascii_downcase=="content-type") | .value, .request.url]
  | @tsv
' "$har_file"

jq -r '
  .log.entries[]
  | select(.response.status == 401 or .response.status == 403 or .response.status == 451)
  | [.response.status, .request.method, .request.url]
  | @tsv
' "$har_file"
```

To inspect the headers for one captured manifest, export the selected URL into the shell and query only that HAR entry:

```bash
export manifest_url="$(pbpaste)"

jq -r '
  .log.entries[]
  | select(.request.url == env.manifest_url)
  | .request.headers[]?
  | select(.name | test("^(referer|origin|user-agent|cookie)$"; "i"))
  | "\(.name): \(.value)"
' "$har_file"
```

## 4. Stream Formats and CDN Analysis

The source-backed CamSoda path is live HLS. The `yt-dlp` extractor passes `live=True` when extracting the `.m3u8` formats, so implementations should not expect a complete VOD-style playlist with a stable `#EXT-X-ENDLIST`.

### HLS Characteristics

Expect:

```text
index.m3u8
#EXT-X-MEDIA-SEQUENCE
#EXT-X-TARGETDURATION
rolling segment window
live=True extraction
token query parameter
```

Live HLS manifests can change while recording. Segments may become unavailable after they roll out of the live playlist window. RFC 8216 describes live playlists where media sequence numbers advance and old segments can be removed, so a downloader should start processing promptly after obtaining a valid manifest.

### CDN Handling

The extractor obtains edge servers from the API response. Store the observed edge hostname for that active session only. Do not treat it as a global CamSoda CDN list, and do not try random hostnames when the API reports offline or private.

### Format Choice

For live HLS, prefer the best available HLS variant unless the user explicitly chooses bandwidth limits. Use `yt-dlp -F` first to inspect available variants.


### Transport Classification and Header Validation

Classify the CamSoda media request before choosing a tool. HLS usually exposes `.m3u8` playlists. A master playlist contains variant metadata such as `#EXT-X-STREAM-INF`; a media playlist contains segment timing such as `#EXTINF` and live sequence tags such as `#EXT-X-MEDIA-SEQUENCE`. A live HLS playlist often lacks `#EXT-X-ENDLIST`, which means more segments can be appended and old segments may disappear as the live window moves. DASH commonly exposes an `.mpd` manifest with fragmented media segments such as `.m4s`. A direct media file exposes a container such as MP4 or WebM. WebRTC and EME-protected playback should be reported as unsupported for URL downloaders.

Validate the captured URL with the same headers seen in the browser. Use HEAD first when the server supports it; if HEAD is rejected but playback works, compare with the browser request method rather than guessing a new URL.

```bash
page_url="$(pbpaste)"
origin="https://camsoda.com"
ua="Mozilla/5.0"
manifest_url="$(pbpaste)"

curl -I \
  -H "Referer: $page_url" \
  -H "Origin: $origin" \
  -H "User-Agent: $ua" \
  "$manifest_url"

curl -L --range 0-2047 \
  -H "Referer: $page_url" \
  -H "Origin: $origin" \
  -H "User-Agent: $ua" \
  "$manifest_url"
```

For HLS, read the playlist text and check whether it is a master playlist, a media playlist, or an error document returned with a media-looking URL:

```bash
curl -L \
  -H "Referer: $page_url" \
  -H "Origin: $origin" \
  -H "User-Agent: $ua" \
  "$manifest_url" \
  | sed -n '1,40p'
```

Red flags in the validation output:

- HTML login page returned instead of a playlist
- `401` or `403` after copying a manifest without cookies or required headers
- `404` or `410` after a delay, indicating an expired signed URL or ended live session
- `451`, region text, or geo-specific denial response
- License, key-system, `MediaKey`, or `encrypted` events that indicate EME/DRM handling
- WebSocket/WebRTC traffic without an HLS, DASH, or direct media fallback

## 5. yt-dlp Implementation Strategies

CamSoda has dedicated `yt-dlp` support.

### List Formats

The commands below assume `room_url` is the authorized room URL and `manifest_url` is a current HLS URL observed from the authorized session.

```bash
yt-dlp -F "$room_url"
```

### Download an Authorized Public Live Stream

```bash
yt-dlp \
  --hls-use-mpegts \
  -o "camsoda-%(id)s-%(epoch)s.%(ext)s" \
  "$room_url"
```

`--hls-use-mpegts` is useful for live recordings because interrupted live downloads can be easier to recover when the intermediate container is MPEG-TS.

### Use Browser Cookies When Needed

```bash
yt-dlp \
  --cookies-from-browser chrome \
  --add-headers "Referer:$room_url" \
  "$room_url"
```

### Limit Recording Duration

```bash
yt-dlp \
  --download-sections "*00:00:00-00:10:00" \
  "$room_url"
```

For live streams, duration behavior can vary by extractor and downloader path. If duration limiting is critical, use FFmpeg's `-t` after obtaining an authorized manifest.

### Inspect JSON Metadata

```bash
yt-dlp \
  --dump-json \
  "$room_url" | jq .
```

Expected failure messages should be surfaced directly to users:

```text
Model is in private show.
Model is offline.
No active streams found.
```


### yt-dlp Session Commands

Local yt-dlp 2025.12.08 lists a `Camsoda` extractor, so a first pass can test the page URL with `--list-formats`. Keep the browser-captured manifest path available because live room state and authorization can still make the generic page attempt fail.

Start with metadata and format listing. These commands keep the browser session in scope and avoid inventing media URLs:

```bash
site_url="$(pbpaste)"
origin="https://camsoda.com"
ua="Mozilla/5.0"

yt-dlp \
  --cookies-from-browser chrome \
  --add-headers "Referer:$site_url" \
  --add-headers "Origin:$origin" \
  --user-agent "$ua" \
  --list-formats \
  "$site_url"

yt-dlp \
  --cookies-from-browser chrome \
  --add-headers "Referer:$site_url" \
  --add-headers "Origin:$origin" \
  --user-agent "$ua" \
  --dump-json \
  --simulate \
  "$site_url" \
  | jq '{extractor, id, title, live_status, is_live, availability, formats: (.formats | length)}'
```

If DevTools or the extension captured an HLS/DASH manifest from an authorized CamSoda session, pass that exact manifest URL instead of constructing one:

```bash
manifest_url="$(pbpaste)"

yt-dlp \
  --cookies-from-browser chrome \
  --add-headers "Referer:$site_url" \
  --add-headers "Origin:$origin" \
  --user-agent "$ua" \
  --downloader ffmpeg \
  --merge-output-format mp4 \
  --output "camsoda-com-%(epoch)s.%(ext)s" \
  "$manifest_url"
```

For diagnostics, write a verbose log in the project working directory and inspect the structured fields with `jq`:

```bash
yt-dlp \
  --verbose \
  --cookies-from-browser chrome \
  --add-headers "Referer:$site_url" \
  --add-headers "Origin:$origin" \
  --user-agent "$ua" \
  --dump-json \
  --simulate \
  "$manifest_url" \
  > "camsoda-com-yt-dlp-info.json"

jq '{extractor, protocol, http_headers, requested_formats, formats: [.formats[]? | {format_id, protocol, ext, height, tbr}]}' \
  "camsoda-com-yt-dlp-info.json"
```

Do not treat a failed extractor as proof that CamSoda has no playable stream. For live/auth pages, it often means the extractor did not receive the browser-only state, the room changed status, the signed URL expired, or the transport is not a normal URL-based stream.

## 6. FFmpeg Processing Techniques

Use FFmpeg when you already have an authorized HLS manifest. FFmpeg does not replace the room-state API or permission checks.

### Record a Valid Live HLS Manifest

```bash
ffmpeg \
  -headers "Referer: $room_url"$'\n'"User-Agent: Mozilla/5.0"$'\n' \
  -i "$manifest_url" \
  -t 00:10:00 \
  -c copy \
  -bsf:a aac_adtstoasc \
  "camsoda-live.mp4"
```

### More Resilient Live Capture

```bash
ffmpeg \
  -reconnect 1 \
  -reconnect_streamed 1 \
  -reconnect_delay_max 5 \
  -headers "Referer: $room_url"$'\n'"User-Agent: Mozilla/5.0"$'\n' \
  -i "$manifest_url" \
  -c copy \
  "camsoda-live.ts"
```

### Remux After Capture

```bash
ffmpeg \
  -i "camsoda-live.ts" \
  -c copy \
  -bsf:a aac_adtstoasc \
  "camsoda-live.mp4"
```

If the room goes offline or the token expires, refresh through the authorized page/API flow rather than editing the token.


### ffprobe and FFmpeg Commands

Use `ffprobe` before a full capture. It confirms whether the captured CamSoda URL is a real media resource and whether the browser headers are sufficient:

```bash
header_block=$'Referer: '"$site_url"$'\r\nOrigin: '"$origin"$'\r\nUser-Agent: '"$ua"$'\r\n'

ffprobe \
  -v error \
  -headers "$header_block" \
  -show_entries format=format_name,duration:stream=index,codec_name,codec_type,width,height,avg_frame_rate \
  -of json \
  "$manifest_url" \
  | jq .
```

For a short authorized HLS capture, keep the first run small and use stream copy. This tests request continuity, segment access, timestamp handling, and output muxing without committing to a long live recording:

```bash
ffmpeg \
  -rw_timeout 15000000 \
  -headers "$header_block" \
  -i "$manifest_url" \
  -t 00:02:00 \
  -c copy \
  -bsf:a aac_adtstoasc \
  "camsoda-com-authorized-sample.mp4"
```

For live playlists that move quickly, MPEG-TS can be a practical intermediate container because it is tolerant of live segment boundaries. Remux afterward only if the sample plays correctly:

```bash
ffmpeg \
  -rw_timeout 15000000 \
  -headers "$header_block" \
  -i "$manifest_url" \
  -t 00:05:00 \
  -c copy \
  "camsoda-com-authorized-capture.ts"

ffmpeg \
  -i "camsoda-com-authorized-capture.ts" \
  -c copy \
  -bsf:a aac_adtstoasc \
  "camsoda-com-authorized-capture.mp4"
```

If the browser request included a `Cookie` header that `--cookies-from-browser` cannot reproduce for FFmpeg, copy it only for the current diagnostic session and avoid saving it in shell history or committed files:

```bash
cookie_header="$(pbpaste)"
header_block=$'Referer: '"$site_url"$'\r\nOrigin: '"$origin"$'\r\nUser-Agent: '"$ua"$'\r\nCookie: '"$cookie_header"$'\r\n'
```

Stop and report unsupported when the validation points to EME/DRM, WebRTC-only transport, unavailable entitlement, or a private state the current browser session cannot play.

## 7. Alternative Tools and Backup Methods

### Browser Network Inspection

Filter the network log for:

```text
vtoken
index.m3u8
token=
stream_name
edge_servers
```

This is useful when debugging extension behavior, but the product should avoid requiring users to inspect DevTools manually.

### Streamlink

Streamlink can test a valid HLS URL:

```bash
streamlink \
  --http-header "Referer=$room_url" \
  "$manifest_url" best
```

### VLC or mpv

Use a desktop player to confirm whether the observed HLS URL plays with the same headers. Failure outside the browser usually means missing headers, expired token, or access state changes.

### curl

Headers-only checks can help diagnose expiry:

```bash
curl -I \
  -H "Referer: $room_url" \
  "$manifest_url"
```

Do not repeatedly poll private or offline rooms.

## 8. Implementation Recommendations

For CamSoda, the extension should:

- Detect room URLs on `www.camsoda.com` where the path is the username.
- Let `yt-dlp` handle supported extraction when possible.
- Preserve explicit failure states for private and offline rooms.
- Treat HLS tokens as short-lived session data.
- Avoid hard-coded edge hostnames.
- Start live captures quickly after manifest discovery.
- Provide a recording-duration option for live streams.
- Avoid claiming support for private shows, paid access controls, or hidden rooms.

Recommended user-facing messages:

```text
This room is offline.
This room is in a private show and cannot be downloaded.
The stream token expired. Refresh the page and try again.
No active HLS formats were found.
```


### Implementation Blueprint for CamSoda

A robust CamSoda downloader should be a detector and classifier before it is a downloader. The browser extension path should collect facts in this order:

1. Confirm that the active tab host matches `camsoda.com` or an expected first-party subdomain.
2. Confirm that the user can play the target in the browser session.
3. Capture the final media request after redirects, not just the visible page URL.
4. Store the minimal request context needed for the current attempt: page URL, manifest URL, user-agent, referer, origin, and current cookies.
5. Classify the transport as HLS, DASH, direct media, WebRTC, EME/DRM, offline, geo-blocked, missing entitlement, or unknown.
6. Pass URL-based HLS/DASH/direct media to `yt-dlp`, `ffprobe`, or FFmpeg with the observed headers.
7. Display a precise unsupported state instead of retrying guessed CDN URLs.

A conservative failure taxonomy makes support easier:

```text
playback_not_started
room_or_post_offline
login_required
missing_entitlement
private_or_paid_state
geo_or_age_restricted
manifest_not_found
manifest_expired
headers_missing
cookie_context_missing
hls_supported
dash_supported
direct_media_supported
webrtc_only_unsupported
drm_or_eme_unsupported
extractor_changed
```

That taxonomy keeps the implementation honest. The extension can help users save streams they can already access, but it should not imply private-room access, subscription bypass, DRM bypass, account sharing, or fixed CDN URL construction.

## 9. Troubleshooting and Edge Cases

### Room is offline

This is an expected state. Wait until the room is public and live.

### Private show detected

Do not proceed. The extractor reports private shows as expected failures.

### No active streams found

The API may have returned stale or incomplete stream data. Refresh the room page and retry once; then report the state.

### `403 Forbidden` on HLS

The token may have expired, or the request may be missing referer/user-agent/session headers. Re-capture a fresh authorized manifest.

### Recording starts late

Live HLS only exposes the current rolling window. Start capture as soon as a valid manifest is available.

### Output is corrupted after interruption

Use `--hls-use-mpegts` with `yt-dlp`, or capture to `.ts` and remux to MP4 after the recording ends.


### Additional Troubleshooting Checks

**The manifest worked in the browser but fails in FFmpeg.** Compare request headers from the HAR. In live/auth pages, `Referer`, `Origin`, `User-Agent`, cookies, and signed query strings can all matter. Re-capture after playback starts instead of reusing an old URL.

**The HAR shows HLS but no complete file appears.** Live HLS playlists may omit `#EXT-X-ENDLIST` and remove older segments as the window advances. Use a short test capture first, then record while the stream is live.

**The response is HTML, not media.** The URL may have redirected to login, age verification, geo restriction, or an error page. Check status code, content type, and response size before passing it to `yt-dlp` or FFmpeg.

**The player uses `blob:` URLs.** A `blob:` URL is usually a browser-local object URL, not a downloadable network URL. Go back to the Network panel and find the underlying HLS, DASH, direct file, WebRTC, or EME path.

**The page exposes DASH but FFmpeg reports encryption or missing keys.** Report unsupported unless the browser-accessible manifest and media segments are clear, authorized, and not DRM protected.

**The capture fails after a few minutes.** Re-check signed URL lifetime, CDN cookie lifetime, and whether the live room changed state. Long-running live captures should refresh only by re-detecting from the authorized page, not by guessing URL parameters.

## 10. Sources

- [yt-dlp README and command options](https://github.com/yt-dlp/yt-dlp/blob/master/README.md)
- [yt-dlp supported sites](https://github.com/yt-dlp/yt-dlp/blob/master/supportedsites.md)
- [yt-dlp CamSoda extractor source](https://github.com/yt-dlp/yt-dlp/blob/master/yt_dlp/extractor/camsoda.py)
- [FFmpeg main documentation](https://ffmpeg.org/ffmpeg.html)
- [FFmpeg protocol documentation for HTTP headers, user-agent, and referer](https://ffmpeg.org/ffmpeg-protocols.html)
- [ffprobe documentation](https://ffmpeg.org/ffprobe.html)
- [RFC 8216: HTTP Live Streaming](https://www.rfc-editor.org/rfc/rfc8216)
- [Chrome DevTools Network features reference](https://developer.chrome.com/docs/devtools/network/reference/)
- [jq manual](https://jqlang.org/manual/)
- [MDN Media Source API](https://developer.mozilla.org/en-US/docs/Web/API/Media_Source_Extensions_API)
- [MDN Encrypted Media Extensions API](https://developer.mozilla.org/en-US/docs/Web/API/Encrypted_Media_Extensions_API)
- Local tool verification: `yt-dlp 2025.12.08`, `ffmpeg`, `ffprobe`, and `jq` were present in the local environment; extractor names were checked locally without opening CamSoda media pages.
