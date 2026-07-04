# How to Download FanslyLive Videos | Technical Analysis of Live Authorization, Stream Patterns, CDNs, and Download Methods

This research document provides a technical analysis of FanslyLive video delivery, including live/offline state, subscription tiers, post permissions, browser cookies, short-lived manifests, and safe command-line handling. Fansly Live is designed around real-time streams, and Fansly content can be gated by follower, subscriber, tier, or purchase state, so a downloader must preserve authorization context and avoid implying any bypass of platform controls.

But first...

## FanslyLive Video Downloader Chrome Extension

If you prefer a simpler one-click method, check out the FanslyLive Video Downloader browser extension.

FanslyLive Downloader is a browser extension built for users who want a cleaner way to save accessible FanslyLive videos without falling back to developer tools, network tabs, or command-line utilities. It detects supported media sources in the active browser session and exports the final result as a usable local file for offline playback.

- Save supported FanslyLive videos from pages you are authorized to access
- Detect available media sources without manual network inspection
- Preserve browser-session context where cookies, referers, or signed URLs are required
- Export detected video as a standard local media file when supported
- Use a browser-native workflow instead of separate desktop tools

👉 Click here to try it free: https://serp.ly/fanslylive-downloader?via=github&utm_source=github&utm_medium=piggyback&utm_campaign=howtodownloadvideosagent

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [FanslyLive Video Infrastructure Overview](#2-fanslylive-video-infrastructure-overview)
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

FanslyLive extraction is an authorization and live-state problem. Fansly's help center documents Fansly Live as a real-time streaming feature, and its creator documentation describes followers, subscribers, subscription tiers, locked permissions, and paid access models. That means media detection must answer two questions before attempting a download:

1. Is a supported media source available in this browser session?
2. Is the viewer authorized for the post, subscription tier, live stream, or paid content that generated it?

This research did not visit explicit media pages and did not download site media. Local tool checks showed:

```bash
yt-dlp 2025.12.08
ffmpeg version 8.0.1
```

Local `yt-dlp --list-extractors` did not show a dedicated Fansly or FanslyLive extractor. Use provider-specific claims cautiously. `yt-dlp` can still be useful for generic pages or authorized direct media URLs, but the browser session is the source of truth.

## 2. FanslyLive Video Infrastructure Overview

Fansly content should be modeled as a JavaScript application with explicit permission gates. A user may be a follower, subscriber, subscriber in a specific tier, expired subscriber, purchaser of a media bundle, or neither. A page can render while the actual media remains unavailable until the proper entitlement is present.

### Access Layers

Separate the workflow into:

1. Application access: the page or route loads.
2. Permission access: the user can view the post, media bundle, or live stream.
3. Player access: the browser can request player configuration or stream metadata.
4. Media access: the final manifest, segment, or direct file URL is requestable.

Each layer can fail independently. A downloader should not collapse every failure into "download failed."

### Evidence to Search

Search the rendered app and network log for:

```text
.m3u8
.mpd
.m4s
.ts
.mp4
.webm
blob:
MediaSource
SourceBuffer
live
stream
manifest
playlist
permission
subscription
subscriber
tier
bundle
token
signature
expires
```

### State Model

```text
authorized_live
authorized_vod
offline
not_subscribed
wrong_subscription_tier
purchase_required
creator_or_platform_removed_media
expired_manifest
cookie_or_header_required
unsupported_or_drm
```

Do not attempt to bypass permissions, subscription tiers, paid bundles, or account blocks.


### Browser Session and Authorization Model

For FanslyLive, the reliable implementation boundary is the same boundary the browser enforces. The downloader should begin from a page the user can already play, then carry forward only the request context needed for that current session. That context usually includes the page URL, the request URL for the manifest or direct media object, the browser user-agent, cookies scoped to the site, and the `Referer` and `Origin` headers observed on the media request. Do not hard-code CDN hosts or construct media URLs from usernames, room names, post IDs, or visible page slugs.

A conservative capture record should include only the active FanslyLive page URL, the final authorized manifest or direct-media URL, the observed `Referer`, `Origin`, `User-Agent`, and cookie context, the HTTP status and content type, the detected transport, and the capture time for the current diagnostic session. The expected origin for first-party browser requests is `https://fansly.com` unless the HAR proves a different first-party origin for the active page.

The capture should expire with the browser session. Signed query strings, CDN cookies, and entitlement-bearing headers may be valid only for a short time. A useful extension does not try alternate private URLs after a failure; it refreshes the page, re-detects the active media request, and reports the exact authorization or transport state.

Implementation detail that is worth logging:

- Whether playback was actually started before detection
- Whether the room, post, purchase, or clip is public, authenticated, paid, private, offline, or geo-blocked
- Whether the media request returned `200`, `206`, `302`, `401`, `403`, `404`, `410`, or `451`
- Whether the response content type looked like HLS, DASH, MP4/WebM, JavaScript configuration, JSON API metadata, or a license/key request
- Whether the final media URL contained an expiry, signature, policy, token, or CDN cookie dependency
- Whether browser APIs indicated Media Source Extensions, Encrypted Media Extensions, or WebRTC instead of a plain downloadable URL

## 3. Embed URL Patterns and Detection

### Host-Level Detection

Fansly is an application site, so URL shape should be treated as a routing hint rather than final media evidence.

```regex
https?:\/\/(?:www\.)?fansly\.com\/[^\s"'<>]+
```

Use this to identify candidate pages, then inspect browser-rendered state and network requests for actual media. Avoid relying on a single post or profile route shape because SPA routes can change.

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
    /manifest|playlist|stream|live|media|permission|subscription|subscriber|tier|bundle|token|signature|expires/i.test(url)
  );
```

### Preserve Request Context

Capture:

- Final Fansly page URL
- Active user cookies
- Referer, origin, and user-agent
- Full manifest or direct file URL
- All signed query parameters
- Permission state shown by the UI or API response
- Time of discovery

Fansly Live streams should be treated as ephemeral. If the stream ends, the live manifest may no longer be useful.


### Browser, HAR, and Request Inspection Workflow

Use DevTools only after the FanslyLive player has begun playback. Chrome DevTools records network requests while the Network panel is open, and the Preserve log setting keeps requests visible across navigation. For live/auth sites, that matters because the manifest request may appear after a room-state API call, a player bootstrap request, or a redirect.

Practical browser checks:

1. Open the FanslyLive page that already plays in the browser.
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
har_file="$PWD/fansly-com-session.har"

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

Because no dedicated local extractor was found, format discovery should come from the active page.

### HLS

HLS `.m3u8` manifests are common for web live and adaptive VOD delivery. In a live context, expect rolling playlists, changing media sequence numbers, and short segment availability. Preserve the exact manifest URL and headers.

### DASH

DASH `.mpd` manifests and `.m4s` segments can appear behind Media Source Extensions. If the visible video element shows `blob:`, the actual media source is in network requests, not the element `src`.

### Direct MP4 or WebM

Direct files can appear for uploaded VOD or processed media. They may still require cookies or signed URLs. Do not infer alternate qualities by editing the URL.

### DRM

If playback involves a license server or Encrypted Media Extensions, the downloader should report DRM or unsupported playback. Do not attempt to bypass content protection.

### CDN Handling

Do not invent Fansly CDN domains. Store only the hostnames and URLs observed in the current authorized session. A failed media URL should trigger re-detection through the browser, not hostname probing.


### Transport Classification and Header Validation

Classify the FanslyLive media request before choosing a tool. HLS usually exposes `.m3u8` playlists. A master playlist contains variant metadata such as `#EXT-X-STREAM-INF`; a media playlist contains segment timing such as `#EXTINF` and live sequence tags such as `#EXT-X-MEDIA-SEQUENCE`. A live HLS playlist often lacks `#EXT-X-ENDLIST`, which means more segments can be appended and old segments may disappear as the live window moves. DASH commonly exposes an `.mpd` manifest with fragmented media segments such as `.m4s`. A direct media file exposes a container such as MP4 or WebM. WebRTC and EME-protected playback should be reported as unsupported for URL downloaders.

Validate the captured URL with the same headers seen in the browser. Use HEAD first when the server supports it; if HEAD is rejected but playback works, compare with the browser request method rather than guessing a new URL.

```bash
page_url="$(pbpaste)"
origin="https://fansly.com"
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

FanslyLive does not have dedicated local `yt-dlp` extractor support. Use `yt-dlp` for generic extraction tests or authorized direct media URLs.

### Test the Current Page

```bash
yt-dlp -F "$page_url"
```

### Include Browser Cookies

```bash
yt-dlp \
  --cookies-from-browser chrome \
  --add-headers "Referer:$page_url" \
  -F "$page_url"
```

### Download an Authorized HLS Manifest

```bash
yt-dlp \
  --hls-use-mpegts \
  --add-headers "Referer:$page_url" \
  -o "fanslylive-%(epoch)s.%(ext)s" \
  "$manifest_url"
```

### Download an Authorized Direct File

```bash
yt-dlp \
  --add-headers "Referer:$page_url" \
  -o "fanslylive-%(epoch)s.%(ext)s" \
  "$direct_file_url"
```

### Metadata Probe

```bash
yt-dlp \
  --cookies-from-browser chrome \
  --dump-json \
  "$page_url" | jq .
```

If generic extraction fails, use browser network inspection. Do not treat lack of `yt-dlp` provider support as permission to bypass authorization.


### yt-dlp Session Commands

Local yt-dlp 2025.12.08 did not list a dedicated FanslyLive extractor in the checked extractor names. Start with browser-session evidence and captured manifests instead of assuming page-URL support.

Start with metadata and format listing. These commands keep the browser session in scope and avoid inventing media URLs:

```bash
site_url="$(pbpaste)"
origin="https://fansly.com"
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

If DevTools or the extension captured an HLS/DASH manifest from an authorized FanslyLive session, pass that exact manifest URL instead of constructing one:

```bash
manifest_url="$(pbpaste)"

yt-dlp \
  --cookies-from-browser chrome \
  --add-headers "Referer:$site_url" \
  --add-headers "Origin:$origin" \
  --user-agent "$ua" \
  --downloader ffmpeg \
  --merge-output-format mp4 \
  --output "fansly-com-%(epoch)s.%(ext)s" \
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
  > "fansly-com-yt-dlp-info.json"

jq '{extractor, protocol, http_headers, requested_formats, formats: [.formats[]? | {format_id, protocol, ext, height, tbr}]}' \
  "fansly-com-yt-dlp-info.json"
```

Do not treat a failed extractor as proof that FanslyLive has no playable stream. For live/auth pages, it often means the extractor did not receive the browser-only state, the room changed status, the signed URL expired, or the transport is not a normal URL-based stream.

## 6. FFmpeg Processing Techniques

FFmpeg is useful once an authorized manifest or direct file URL has been captured.

### Record an Authorized Live HLS Stream

```bash
ffmpeg \
  -headers "Referer: $page_url"$'\n'"User-Agent: Mozilla/5.0"$'\n'"Cookie: $cookie_header"$'\n' \
  -i "$manifest_url" \
  -t 00:10:00 \
  -c copy \
  -bsf:a aac_adtstoasc \
  "fanslylive-recording.mp4"
```

### Save an Authorized VOD Manifest

```bash
ffmpeg \
  -headers "Referer: $page_url"$'\n'"User-Agent: Mozilla/5.0"$'\n'"Cookie: $cookie_header"$'\n' \
  -i "$manifest_url" \
  -c copy \
  -bsf:a aac_adtstoasc \
  "fanslylive-video.mp4"
```

### Probe Before Recording

```bash
ffprobe \
  -headers "Referer: $page_url"$'\n'"User-Agent: Mozilla/5.0"$'\n'"Cookie: $cookie_header"$'\n' \
  "$manifest_url"
```

If FFmpeg reports missing segments, the stream may have ended or the manifest may have expired.


### ffprobe and FFmpeg Commands

Use `ffprobe` before a full capture. It confirms whether the captured FanslyLive URL is a real media resource and whether the browser headers are sufficient:

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
  "fansly-com-authorized-sample.mp4"
```

For live playlists that move quickly, MPEG-TS can be a practical intermediate container because it is tolerant of live segment boundaries. Remux afterward only if the sample plays correctly:

```bash
ffmpeg \
  -rw_timeout 15000000 \
  -headers "$header_block" \
  -i "$manifest_url" \
  -t 00:05:00 \
  -c copy \
  "fansly-com-authorized-capture.ts"

ffmpeg \
  -i "fansly-com-authorized-capture.ts" \
  -c copy \
  -bsf:a aac_adtstoasc \
  "fansly-com-authorized-capture.mp4"
```

If the browser request included a `Cookie` header that `--cookies-from-browser` cannot reproduce for FFmpeg, copy it only for the current diagnostic session and avoid saving it in shell history or committed files:

```bash
cookie_header="$(pbpaste)"
header_block=$'Referer: '"$site_url"$'\r\nOrigin: '"$origin"$'\r\nUser-Agent: '"$ua"$'\r\nCookie: '"$cookie_header"$'\r\n'
```

Stop and report unsupported when the validation points to EME/DRM, WebRTC-only transport, unavailable entitlement, or a private state the current browser session cannot play.

## 7. Alternative Tools and Backup Methods

### Browser Extension

The browser extension path is preferable because it runs in the authorized session and can see media requests after playback starts.

### HAR Export

Use a HAR export only for debugging your own session. HAR files can contain cookies, tokens, and signed URLs, so they should not be shared.

### Browser DevTools

Filter the Network panel for `m3u8`, `mpd`, `m4s`, `ts`, `mp4`, and `webm`. Start playback before filtering.

### VLC, IINA, or Streamlink

These tools can help with a valid manifest, but they usually need extra headers and cookies for gated pages. They do not solve subscription or permission checks.

## 8. Implementation Recommendations

1. Run detection inside the browser session, not from unauthenticated server requests.
2. Classify live, offline, subscribed, expired, purchase-required, and unsupported states separately.
3. Preserve all cookies, headers, and query parameters for media requests.
4. Never imply support for subscription bypass, tier bypass, paid bundle bypass, private rooms, or DRM.
5. Prefer manifest-level downloads over segment-level downloads.
6. Re-detect manifests when a live stream changes state.
7. Keep error messages explicit and user-facing.
8. Avoid storing cookies or signed media URLs longer than necessary.


### Implementation Blueprint for FanslyLive

A robust FanslyLive downloader should be a detector and classifier before it is a downloader. The browser extension path should collect facts in this order:

1. Confirm that the active tab host matches `fansly.com` or an expected first-party subdomain.
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

### The Live Stream Is Offline

Report `offline`. A live URL cannot be constructed when no current stream exists.

### The Viewer Is Not in the Required Tier

Report the subscription or purchase requirement. Do not retry with alternate routes.

### The Video Element Is `blob:`

Inspect network requests for manifests and segments. A blob URL is not the original media URL.

### yt-dlp Does Not Recognize the Page

That is expected because no dedicated local Fansly extractor was found. Try the authorized manifest URL if one is visible.

### The Manifest Expires

Reload the page, start playback, and detect a fresh manifest from the same authorized session.

### DRM Is Detected

Report unsupported. FFmpeg and `yt-dlp` should not be used to bypass DRM.


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
- [Fansly Streaming Content Guidelines](https://help.fansly.com/en/articles/10544521-streaming-content-guidelines)
- [Fansly followers and subscribers documentation](https://help.fansly.com/hc/en-us/articles/27666080601107-Your-audience-on-Fansly-Followers-and-Subscribers)
- [Fansly Terms](https://fansly.com/terms)
- [FFmpeg main documentation](https://ffmpeg.org/ffmpeg.html)
- [FFmpeg protocol documentation for HTTP headers, user-agent, and referer](https://ffmpeg.org/ffmpeg-protocols.html)
- [ffprobe documentation](https://ffmpeg.org/ffprobe.html)
- [RFC 8216: HTTP Live Streaming](https://www.rfc-editor.org/rfc/rfc8216)
- [Chrome DevTools Network features reference](https://developer.chrome.com/docs/devtools/network/reference/)
- [jq manual](https://jqlang.org/manual/)
- [MDN Media Source API](https://developer.mozilla.org/en-US/docs/Web/API/Media_Source_Extensions_API)
- [MDN Encrypted Media Extensions API](https://developer.mozilla.org/en-US/docs/Web/API/Encrypted_Media_Extensions_API)
- Local tool verification: `yt-dlp 2025.12.08`, `ffmpeg`, `ffprobe`, and `jq` were present in the local environment; extractor names were checked locally without opening FanslyLive media pages.
