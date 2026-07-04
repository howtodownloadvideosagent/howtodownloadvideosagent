# How to Download MyFreeCams Videos | Technical Analysis of Live Authorization, Stream Patterns, CDNs, and Download Methods

This research document provides a technical analysis of MyFreeCams video delivery from a live-stream and authorization-first perspective: model room links, profile URLs, live/offline state, tokens, private and group shows, cookies, short-lived media URLs, and safe handling for authorized sessions. MyFreeCams is a live webcam network, so a downloader must avoid implying support for private-room access, token bypass, access-control bypass, or recording without permission.

But first...

## MyFreeCams Video Downloader Chrome Extension

If you prefer a simpler one-click method, check out the MyFreeCams Video Downloader browser extension.

MyFreeCams Downloader is a browser extension built for users who want a cleaner way to save accessible MyFreeCams videos without falling back to developer tools, network tabs, or command-line utilities. It detects supported media sources in the active browser session and exports the final result as a usable local file for offline playback.

- Save supported MyFreeCams videos from pages you are authorized to access
- Detect available media sources without manual network inspection
- Preserve browser-session context where cookies, referers, or signed URLs are required
- Export detected video as a standard local media file when supported
- Use a browser-native workflow instead of separate desktop tools

👉 Click here to try it free: https://serp.ly/myfreecams-downloader?via=github&utm_source=github&utm_medium=piggyback&utm_campaign=howtodownloadvideosagent

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [MyFreeCams Video Infrastructure Overview](#2-myfreecams-video-infrastructure-overview)
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

MyFreeCams extraction is primarily a live-state and authorization problem. Public MyFreeCams wiki documentation describes tokens used for private shows, group shows, spy shows, and tips. That makes a conservative implementation essential: the downloader should distinguish public live playback from token-gated, private, group, spy, blocked, or unsupported states.

This research did not visit explicit media pages and did not download site media. Local tool checks showed:

```bash
yt-dlp 2025.12.08
ffmpeg version 8.0.1
```

Local `yt-dlp --list-extractors` did not show a dedicated MyFreeCams extractor. A downloader should rely on browser-observed media and should report private, group, token-gated, offline, blocked, and unsupported states directly.

## 2. MyFreeCams Video Infrastructure Overview

MyFreeCams should be modeled as a live webcam platform with both public and account-mediated room states.

### Public Documentation Signals

The same public documentation identifies:

- Main site access through `www.myfreecams.com`
- Model profile pages under `profiles.myfreecams.com/{ModelUsername}`
- Direct room links using `www.myfreecams.com/#ModelName`
- Model broadcasting through a browser-based Model Web Broadcaster
- Optional OBS or external broadcaster workflows for models
- Tokens for private, group, spy, and tip interactions

### State Model

```text
public_live
offline
private_show
group_show
spy_show
token_required
not_authorized
geo_or_user_blocked
cookie_or_header_required
expired_manifest
webrtc_or_browser_only_transport
unsupported_or_drm
```

If a room is private, group, spy, or token-gated, the downloader should not attempt to bypass that state.

### Evidence to Search

```text
.m3u8
.mpd
.m4s
.ts
.mp4
blob:
MediaSource
SourceBuffer
RTCPeerConnection
webrtc
manifest
playlist
token
private
group
spy
room
model
expires
signature
```


### Browser Session and Authorization Model

For MyFreeCams, the reliable implementation boundary is the same boundary the browser enforces. The downloader should begin from a page the user can already play, then carry forward only the request context needed for that current session. That context usually includes the page URL, the request URL for the manifest or direct media object, the browser user-agent, cookies scoped to the site, and the `Referer` and `Origin` headers observed on the media request. Do not hard-code CDN hosts or construct media URLs from usernames, room names, post IDs, or visible page slugs.

A conservative capture record should include only the active MyFreeCams page URL, the final authorized manifest or direct-media URL, the observed `Referer`, `Origin`, `User-Agent`, and cookie context, the HTTP status and content type, the detected transport, and the capture time for the current diagnostic session. The expected origin for first-party browser requests is `https://myfreecams.com` unless the HAR proves a different first-party origin for the active page.

The capture should expire with the browser session. Signed query strings, CDN cookies, and entitlement-bearing headers may be valid only for a short time. A useful extension does not try alternate private URLs after a failure; it refreshes the page, re-detects the active media request, and reports the exact authorization or transport state.

Implementation detail that is worth logging:

- Whether playback was actually started before detection
- Whether the room, post, purchase, or clip is public, authenticated, paid, private, offline, or geo-blocked
- Whether the media request returned `200`, `206`, `302`, `401`, `403`, `404`, `410`, or `451`
- Whether the response content type looked like HLS, DASH, MP4/WebM, JavaScript configuration, JSON API metadata, or a license/key request
- Whether the final media URL contained an expiry, signature, policy, token, or CDN cookie dependency
- Whether browser APIs indicated Media Source Extensions, Encrypted Media Extensions, or WebRTC instead of a plain downloadable URL

## 3. Embed URL Patterns and Detection

### Room and Profile URL Patterns

Documented profile pattern:

```regex
https?:\/\/profiles\.myfreecams\.com\/([A-Za-z0-9_]+)(?:[/?#]|$)
```

Documented direct room-link pattern:

```regex
https?:\/\/(?:www\.)?myfreecams\.com\/#([A-Za-z0-9_]+)
```

Generic host pattern:

```regex
https?:\/\/(?:www\.)?myfreecams\.com\/[^\s"'<>]*
```

Hash fragments are not sent to the server in normal HTTP requests, so browser-side detection matters. A downloader should read the current browser URL and rendered app state, not only server responses.

### Media Detection Regexes

```regex
https?:\/\/[^"'\s<>]+\.m3u8[^"'\s<>]*
https?:\/\/[^"'\s<>]+\.mpd[^"'\s<>]*
https?:\/\/[^"'\s<>]+\.(?:mp4|webm)(?:\?[^"'\s<>]*)?
blob:https?:\/\/[^"'\s<>]+
```

### Browser-Side Probe

```js
performance.getEntriesByType("resource")
  .map(entry => entry.name)
  .filter(url =>
    /\.(m3u8|mpd|m4s|ts|mp4|webm)(\?|$)/i.test(url) ||
    /manifest|playlist|stream|room|model|token|private|group|spy|signature|expires|webrtc/i.test(url)
  );
```

### Preserve Request Context

Capture:

- Current browser URL, including hash fragment
- Model username when available
- Account cookies
- Referer, origin, and user-agent
- Exact media URL and query string
- Room state
- Time of discovery


### Browser, HAR, and Request Inspection Workflow

Use DevTools only after the MyFreeCams player has begun playback. Chrome DevTools records network requests while the Network panel is open, and the Preserve log setting keeps requests visible across navigation. For live/auth sites, that matters because the manifest request may appear after a room-state API call, a player bootstrap request, or a redirect.

Practical browser checks:

1. Open the MyFreeCams page that already plays in the browser.
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
har_file="$PWD/myfreecams-com-session.har"

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

Because no dedicated local extractor was found, stream formats must be detected from the authorized browser session.

### HLS

If a `.m3u8` manifest appears, treat it as a current live-session URL. Live HLS playlists can update continuously, remove old segments, and omit `#EXT-X-ENDLIST` until a stream ends.

### DASH

If `.mpd` or `.m4s` requests appear, the player may be using DASH through Media Source Extensions. Preserve headers and cookies.

### Direct Files

Recorded or profile media may expose direct `.mp4` or `.webm` URLs. They can still be permission-bound and should not be redistributed.

### WebRTC

Some live platforms use low-latency browser transports. If the page uses WebRTC without an exposed HLS or DASH manifest, normal `yt-dlp` and FFmpeg workflows may not apply.

### CDN Handling

Do not invent MyFreeCams CDN domains. Use only media hostnames observed in the authorized session and reacquire them if they expire.


### Transport Classification and Header Validation

Classify the MyFreeCams media request before choosing a tool. HLS usually exposes `.m3u8` playlists. A master playlist contains variant metadata such as `#EXT-X-STREAM-INF`; a media playlist contains segment timing such as `#EXTINF` and live sequence tags such as `#EXT-X-MEDIA-SEQUENCE`. A live HLS playlist often lacks `#EXT-X-ENDLIST`, which means more segments can be appended and old segments may disappear as the live window moves. DASH commonly exposes an `.mpd` manifest with fragmented media segments such as `.m4s`. A direct media file exposes a container such as MP4 or WebM. WebRTC and EME-protected playback should be reported as unsupported for URL downloaders.

Validate the captured URL with the same headers seen in the browser. Use HEAD first when the server supports it; if HEAD is rejected but playback works, compare with the browser request method rather than guessing a new URL.

```bash
page_url="$(pbpaste)"
origin="https://myfreecams.com"
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

MyFreeCams does not have dedicated local `yt-dlp` extractor support.

### Generic Page Test

```bash
yt-dlp -F "$page_url"
```

### Authenticated Generic Test

```bash
yt-dlp \
  --cookies-from-browser chrome \
  --add-headers "Referer:$page_url" \
  -F "$page_url"
```

### Authorized HLS Manifest

```bash
yt-dlp \
  --hls-use-mpegts \
  --add-headers "Referer:$page_url" \
  -o "myfreecams-%(epoch)s.%(ext)s" \
  "$manifest_url"
```

### Direct File URL

```bash
yt-dlp \
  --add-headers "Referer:$page_url" \
  -o "myfreecams-%(epoch)s.%(ext)s" \
  "$direct_file_url"
```

Expected user-facing failures:

```text
No supported media URL found.
The model is offline.
The room is private, group, spy, token-gated, or otherwise not authorized.
The detected transport is WebRTC or DRM and is unsupported.
The manifest expired; reload the authorized room and try again.
```


### yt-dlp Session Commands

Local yt-dlp 2025.12.08 did not list a dedicated MyFreeCams extractor in the checked extractor names. Start with browser-session evidence and captured manifests instead of assuming page-URL support.

Start with metadata and format listing. These commands keep the browser session in scope and avoid inventing media URLs:

```bash
site_url="$(pbpaste)"
origin="https://myfreecams.com"
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

If DevTools or the extension captured an HLS/DASH manifest from an authorized MyFreeCams session, pass that exact manifest URL instead of constructing one:

```bash
manifest_url="$(pbpaste)"

yt-dlp \
  --cookies-from-browser chrome \
  --add-headers "Referer:$site_url" \
  --add-headers "Origin:$origin" \
  --user-agent "$ua" \
  --downloader ffmpeg \
  --merge-output-format mp4 \
  --output "myfreecams-com-%(epoch)s.%(ext)s" \
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
  > "myfreecams-com-yt-dlp-info.json"

jq '{extractor, protocol, http_headers, requested_formats, formats: [.formats[]? | {format_id, protocol, ext, height, tbr}]}' \
  "myfreecams-com-yt-dlp-info.json"
```

Do not treat a failed extractor as proof that MyFreeCams has no playable stream. For live/auth pages, it often means the extractor did not receive the browser-only state, the room changed status, the signed URL expired, or the transport is not a normal URL-based stream.

## 6. FFmpeg Processing Techniques

Use FFmpeg only with a currently authorized manifest or direct file URL.

### Record a Live HLS Manifest

```bash
ffmpeg \
  -headers "Referer: $page_url"$'\n'"User-Agent: Mozilla/5.0"$'\n'"Cookie: $cookie_header"$'\n' \
  -i "$manifest_url" \
  -t 00:10:00 \
  -c copy \
  -bsf:a aac_adtstoasc \
  "myfreecams-live.mp4"
```

### Save a Direct File

```bash
ffmpeg \
  -headers "Referer: $page_url"$'\n'"User-Agent: Mozilla/5.0"$'\n'"Cookie: $cookie_header"$'\n' \
  -i "$direct_file_url" \
  -c copy \
  "myfreecams-video.mp4"
```

### Probe Headers and Streams

```bash
ffprobe \
  -headers "Referer: $page_url"$'\n'"User-Agent: Mozilla/5.0"$'\n'"Cookie: $cookie_header"$'\n' \
  "$manifest_url"
```

If the stream fails after a room state change, reacquire the media from the active page.


### ffprobe and FFmpeg Commands

Use `ffprobe` before a full capture. It confirms whether the captured MyFreeCams URL is a real media resource and whether the browser headers are sufficient:

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
  "myfreecams-com-authorized-sample.mp4"
```

For live playlists that move quickly, MPEG-TS can be a practical intermediate container because it is tolerant of live segment boundaries. Remux afterward only if the sample plays correctly:

```bash
ffmpeg \
  -rw_timeout 15000000 \
  -headers "$header_block" \
  -i "$manifest_url" \
  -t 00:05:00 \
  -c copy \
  "myfreecams-com-authorized-capture.ts"

ffmpeg \
  -i "myfreecams-com-authorized-capture.ts" \
  -c copy \
  -bsf:a aac_adtstoasc \
  "myfreecams-com-authorized-capture.mp4"
```

If the browser request included a `Cookie` header that `--cookies-from-browser` cannot reproduce for FFmpeg, copy it only for the current diagnostic session and avoid saving it in shell history or committed files:

```bash
cookie_header="$(pbpaste)"
header_block=$'Referer: '"$site_url"$'\r\nOrigin: '"$origin"$'\r\nUser-Agent: '"$ua"$'\r\nCookie: '"$cookie_header"$'\r\n'
```

Stop and report unsupported when the validation points to EME/DRM, WebRTC-only transport, unavailable entitlement, or a private state the current browser session cannot play.

## 7. Alternative Tools and Backup Methods

### Browser Extension

A browser extension can inspect the active room, preserve the browser session, and report unsupported private, token-gated, WebRTC, or DRM states without asking users to copy sensitive URLs.

### DevTools Network Panel

Start playback, then filter for `m3u8`, `mpd`, `m4s`, `ts`, `mp4`, and `webm`. Include room-state API responses in debugging.

### HAR Export

HAR files can include cookies and signed media URLs. Treat them as sensitive and use them only for debugging.

### Streamlink or VLC

These tools may work with a current authorized HLS URL, but they do not provide MyFreeCams authorization or token access.

## 8. Implementation Recommendations

1. Respect the platform's stated access and recording limits.
2. Detect room links from both profile URLs and hash-based room URLs.
3. Read browser state because hash fragments are client-side.
4. Preserve exact media URLs, cookies, referer, origin, and user-agent.
5. Report private, group, spy, token-required, offline, blocked, DRM, and WebRTC states clearly.
6. Do not invent CDN domains or alternate media paths.
7. Do not store cookies or signed URLs longer than required.
8. Keep the downloader scoped to content the user is authorized to access and permitted to save.


### Implementation Blueprint for MyFreeCams

A robust MyFreeCams downloader should be a detector and classifier before it is a downloader. The browser extension path should collect facts in this order:

1. Confirm that the active tab host matches `myfreecams.com` or an expected first-party subdomain.
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

### The Model Link Uses `#ModelName`

Read the full browser URL. The server will not receive the hash fragment, so command-line fetches can miss the room identifier.

### The Room Is Private, Group, or Spy

Report the state. Do not attempt to bypass token-gated or private-room access.

### No `.m3u8` Appears

The site may use a different transport, require playback to start, or expose no supported media in the current state.

### yt-dlp Does Not Recognize the Page

No dedicated local MyFreeCams extractor was found. Use authorized manifest detection instead.

### FFmpeg Gets Segment Errors

The live playlist may have advanced, the URL may have expired, or headers may be missing.

### DRM or WebRTC Is Detected

Report unsupported for normal `yt-dlp` and FFmpeg workflows.


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
- [MyFreeCams wiki: Instructions and Features](https://wiki.myfreecams.com/wiki/index.php?action=info&title=Instructions_and_Features)
- [FFmpeg main documentation](https://ffmpeg.org/ffmpeg.html)
- [FFmpeg protocol documentation for HTTP headers, user-agent, and referer](https://ffmpeg.org/ffmpeg-protocols.html)
- [ffprobe documentation](https://ffmpeg.org/ffprobe.html)
- [RFC 8216: HTTP Live Streaming](https://www.rfc-editor.org/rfc/rfc8216)
- [Chrome DevTools Network features reference](https://developer.chrome.com/docs/devtools/network/reference/)
- [jq manual](https://jqlang.org/manual/)
- [MDN Media Source API](https://developer.mozilla.org/en-US/docs/Web/API/Media_Source_Extensions_API)
- [MDN Encrypted Media Extensions API](https://developer.mozilla.org/en-US/docs/Web/API/Encrypted_Media_Extensions_API)
- Local tool verification: `yt-dlp 2025.12.08`, `ffmpeg`, `ffprobe`, and `jq` were present in the local environment; extractor names were checked locally without opening MyFreeCams media pages.
