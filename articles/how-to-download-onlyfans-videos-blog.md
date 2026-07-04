# How to Download OnlyFans Videos | Technical Analysis of Live Authorization, Stream Patterns, CDNs, and Download Methods

This research document provides a technical analysis of OnlyFans video delivery from an authorization-first perspective: subscriptions, pay-per-view posts, messages, live or stream-like playback, browser cookies, signed media URLs, short-lived manifests, and unsupported states such as DRM or missing entitlement. The goal is not to bypass subscriptions, paid messages, private content, account restrictions, DRM, or platform access controls.

But first...

## OnlyFans Video Downloader Chrome Extension

If you prefer a simpler one-click method, check out the OnlyFans Video Downloader browser extension.

OnlyFans Downloader is a browser extension built for users who want a cleaner way to save accessible OnlyFans videos without falling back to developer tools, network tabs, or command-line utilities. It detects supported media sources in the active browser session and exports the final result as a usable local file for offline playback.

- Save supported OnlyFans videos from pages you are authorized to access
- Detect available media sources without manual network inspection
- Preserve browser-session context where cookies, referers, or signed URLs are required
- Export detected video as a standard local media file when supported
- Use a browser-native workflow instead of separate desktop tools

👉 Click here to try it free: https://serp.ly/onlyfans-downloader?via=github&utm_source=github&utm_medium=piggyback&utm_campaign=howtodownloadvideosagent

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [OnlyFans Video Infrastructure Overview](#2-onlyfans-video-infrastructure-overview)
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

OnlyFans extraction is primarily an authenticated-browser problem. A profile, post, message, or purchased media item can be visible or hidden depending on subscription status, PPV purchase, creator settings, account state, geographic restrictions, or platform policy. A reliable downloader should detect media only after the authorized browser session has loaded and the user can already play the content.

This research did not visit explicit media pages and did not download site media. Local tool checks showed:

```bash
yt-dlp 2025.12.08
ffmpeg version 8.0.1
```

Local `yt-dlp --list-extractors` did not show a dedicated OnlyFans extractor. For implementation, the active browser session and observed network requests are the reliable evidence.

## 2. OnlyFans Video Infrastructure Overview

OnlyFans should be modeled as a gated creator-content application, not as a public video host.

### Access Layers

1. Route access: the user can open a profile, post, message, or collection route.
2. Entitlement access: the user has the required subscription, PPV purchase, or message unlock.
3. Player access: the app returns a playable media configuration.
4. Media access: the manifest, segment, or direct file URL is requestable with the same cookies and headers.

Each layer can fail independently. A downloader should present the exact state instead of retrying blindly.

### Evidence to Search

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
manifest
playlist
media
post
message
subscription
purchase
ppv
token
signature
expires
license
drm
```

### State Model

```text
authorized_vod
authorized_live_or_event
not_subscribed
ppv_purchase_required
message_unlock_required
creator_removed_media
account_restricted
expired_manifest
cookie_or_header_required
unsupported_or_drm
```

The downloader should not imply that it can unlock content, restore deleted media, reuse expired signatures, or bypass account restrictions.


### Browser Session and Authorization Model

For OnlyFans, the reliable implementation boundary is the same boundary the browser enforces. The downloader should begin from a page the user can already play, then carry forward only the request context needed for that current session. That context usually includes the page URL, the request URL for the manifest or direct media object, the browser user-agent, cookies scoped to the site, and the `Referer` and `Origin` headers observed on the media request. Do not hard-code CDN hosts or construct media URLs from usernames, room names, post IDs, or visible page slugs.

A conservative capture record should include only the active OnlyFans page URL, the final authorized manifest or direct-media URL, the observed `Referer`, `Origin`, `User-Agent`, and cookie context, the HTTP status and content type, the detected transport, and the capture time for the current diagnostic session. The expected origin for first-party browser requests is `https://onlyfans.com` unless the HAR proves a different first-party origin for the active page.

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

OnlyFans is an application with many route types. Use host-level detection first, then inspect rendered state and network activity.

```regex
https?:\/\/(?:www\.)?onlyfans\.com\/[^\s"'<>]+
```

Do not rely on static HTML alone. The useful media data may be loaded by JavaScript after authentication, route navigation, opening a post, opening a message, or pressing play.

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
    /manifest|playlist|media|post|message|subscription|purchase|ppv|token|signature|expires|license|drm/i.test(url)
  );
```

### Preserve Request Context

Capture:

- Final app route
- Active account cookies
- Referer, origin, and user-agent
- Full media URL and query string
- Entitlement state shown by the UI or API response
- Timestamp of detection

Signed media URLs should not be reused across accounts, routes, or long time windows.


### Browser, HAR, and Request Inspection Workflow

Use DevTools only after the OnlyFans player has begun playback. Chrome DevTools records network requests while the Network panel is open, and the Preserve log setting keeps requests visible across navigation. For live/auth sites, that matters because the manifest request may appear after a room-state API call, a player bootstrap request, or a redirect.

Practical browser checks:

1. Open the OnlyFans page that already plays in the browser.
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
har_file="$PWD/onlyfans-com-session.har"

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

Because no dedicated local extractor was found, stream formats must be discovered from the authorized browser session.

### HLS

HLS manifests use `.m3u8` playlists. They may be used for adaptive playback and can include signed segment URLs. For live or event-style playback, segments can roll out of the playlist quickly.

### DASH

DASH manifests use `.mpd` files and fragmented media. A visible `blob:` URL can indicate Media Source Extensions, where the browser app feeds chunks into the video element.

### Direct MP4 or WebM

Some authorized media may expose direct file URLs. These can still be short-lived, cookie-bound, referer-bound, or tied to the account's entitlement.

### DRM

If license-server requests or Encrypted Media Extensions appear, normal `yt-dlp` and FFmpeg workflows should report unsupported. Do not attempt to bypass DRM or license checks.

### CDN Handling

Do not invent OnlyFans CDN domains. Even if a media hostname is visible in one session, treat it as evidence for that session only. Re-detect from the authorized page when a URL expires.


### Transport Classification and Header Validation

Classify the OnlyFans media request before choosing a tool. HLS usually exposes `.m3u8` playlists. A master playlist contains variant metadata such as `#EXT-X-STREAM-INF`; a media playlist contains segment timing such as `#EXTINF` and live sequence tags such as `#EXT-X-MEDIA-SEQUENCE`. A live HLS playlist often lacks `#EXT-X-ENDLIST`, which means more segments can be appended and old segments may disappear as the live window moves. DASH commonly exposes an `.mpd` manifest with fragmented media segments such as `.m4s`. A direct media file exposes a container such as MP4 or WebM. WebRTC and EME-protected playback should be reported as unsupported for URL downloaders.

Validate the captured URL with the same headers seen in the browser. Use HEAD first when the server supports it; if HEAD is rejected but playback works, compare with the browser request method rather than guessing a new URL.

```bash
page_url="$(pbpaste)"
origin="https://onlyfans.com"
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

OnlyFans does not have dedicated local `yt-dlp` extractor support.

### Generic Page Test

```bash
yt-dlp -F "$page_url"
```

This may fail on normal app routes. That is expected.

### Authenticated Generic Test

```bash
yt-dlp \
  --cookies-from-browser chrome \
  --add-headers "Referer:$page_url" \
  -F "$page_url"
```

### Authorized Manifest Download

```bash
yt-dlp \
  --hls-use-mpegts \
  --add-headers "Referer:$page_url" \
  -o "onlyfans-%(epoch)s.%(ext)s" \
  "$manifest_url"
```

### Authorized Direct File Download

```bash
yt-dlp \
  --add-headers "Referer:$page_url" \
  -o "onlyfans-%(epoch)s.%(ext)s" \
  "$direct_file_url"
```

### Inspect Generic Metadata

```bash
yt-dlp \
  --cookies-from-browser chrome \
  --dump-json \
  "$page_url" | jq .
```

If generic extraction fails but the browser can play the media, inspect network requests from the authorized session. Do not use account cookies to access another user's content or content not unlocked by the account.


### yt-dlp Session Commands

Local yt-dlp 2025.12.08 did not list a dedicated OnlyFans extractor in the checked extractor names. Start with browser-session evidence and captured manifests instead of assuming page-URL support.

Start with metadata and format listing. These commands keep the browser session in scope and avoid inventing media URLs:

```bash
site_url="$(pbpaste)"
origin="https://onlyfans.com"
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

If DevTools or the extension captured an HLS/DASH manifest from an authorized OnlyFans session, pass that exact manifest URL instead of constructing one:

```bash
manifest_url="$(pbpaste)"

yt-dlp \
  --cookies-from-browser chrome \
  --add-headers "Referer:$site_url" \
  --add-headers "Origin:$origin" \
  --user-agent "$ua" \
  --downloader ffmpeg \
  --merge-output-format mp4 \
  --output "onlyfans-com-%(epoch)s.%(ext)s" \
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
  > "onlyfans-com-yt-dlp-info.json"

jq '{extractor, protocol, http_headers, requested_formats, formats: [.formats[]? | {format_id, protocol, ext, height, tbr}]}' \
  "onlyfans-com-yt-dlp-info.json"
```

Do not treat a failed extractor as proof that OnlyFans has no playable stream. For live/auth pages, it often means the extractor did not receive the browser-only state, the room changed status, the signed URL expired, or the transport is not a normal URL-based stream.

## 6. FFmpeg Processing Techniques

Use FFmpeg only after detecting an authorized manifest or direct media URL.

### Save an Authorized HLS Manifest

```bash
ffmpeg \
  -headers "Referer: $page_url"$'\n'"User-Agent: Mozilla/5.0"$'\n'"Cookie: $cookie_header"$'\n' \
  -i "$manifest_url" \
  -c copy \
  -bsf:a aac_adtstoasc \
  "onlyfans-video.mp4"
```

### Limit a Live or Event Recording

```bash
ffmpeg \
  -headers "Referer: $page_url"$'\n'"User-Agent: Mozilla/5.0"$'\n'"Cookie: $cookie_header"$'\n' \
  -i "$manifest_url" \
  -t 00:10:00 \
  -c copy \
  -bsf:a aac_adtstoasc \
  "onlyfans-live-segment.mp4"
```

### Save a Direct File

```bash
ffmpeg \
  -headers "Referer: $page_url"$'\n'"User-Agent: Mozilla/5.0"$'\n'"Cookie: $cookie_header"$'\n' \
  -i "$direct_file_url" \
  -c copy \
  "onlyfans-direct.mp4"
```

### Probe Before Processing

```bash
ffprobe \
  -headers "Referer: $page_url"$'\n'"User-Agent: Mozilla/5.0"$'\n'"Cookie: $cookie_header"$'\n' \
  "$manifest_url"
```


### ffprobe and FFmpeg Commands

Use `ffprobe` before a full capture. It confirms whether the captured OnlyFans URL is a real media resource and whether the browser headers are sufficient:

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
  "onlyfans-com-authorized-sample.mp4"
```

For live playlists that move quickly, MPEG-TS can be a practical intermediate container because it is tolerant of live segment boundaries. Remux afterward only if the sample plays correctly:

```bash
ffmpeg \
  -rw_timeout 15000000 \
  -headers "$header_block" \
  -i "$manifest_url" \
  -t 00:05:00 \
  -c copy \
  "onlyfans-com-authorized-capture.ts"

ffmpeg \
  -i "onlyfans-com-authorized-capture.ts" \
  -c copy \
  -bsf:a aac_adtstoasc \
  "onlyfans-com-authorized-capture.mp4"
```

If the browser request included a `Cookie` header that `--cookies-from-browser` cannot reproduce for FFmpeg, copy it only for the current diagnostic session and avoid saving it in shell history or committed files:

```bash
cookie_header="$(pbpaste)"
header_block=$'Referer: '"$site_url"$'\r\nOrigin: '"$origin"$'\r\nUser-Agent: '"$ua"$'\r\nCookie: '"$cookie_header"$'\r\n'
```

Stop and report unsupported when the validation points to EME/DRM, WebRTC-only transport, unavailable entitlement, or a private state the current browser session cannot play.

## 7. Alternative Tools and Backup Methods

### Browser Extension

A browser extension is the most practical workflow because it can run in the same authenticated app state and detect actual media after the user opens the content.

### Browser DevTools

Use the Network panel after opening the post, message, or media viewer and pressing play. Filter for `m3u8`, `mpd`, `m4s`, `ts`, `mp4`, and `webm`.

### HAR Export

HAR files can contain sensitive cookies, authorization headers, and signed URLs. Use them only for local debugging.

### VLC, IINA, or Streamlink

These tools can process a valid authorized manifest but do not solve OnlyFans authentication, PPV, subscription, message unlocks, or DRM.

## 8. Implementation Recommendations

1. Require an active browser session for detection.
2. Separate app route access from media entitlement.
3. Preserve cookies, referer, origin, user-agent, and full signed query strings.
4. Do not hard-code or invent CDN domains.
5. Treat manifest URLs as short-lived and account-specific.
6. Show explicit states for subscription required, PPV required, message unlock required, deleted media, expired URL, DRM, and unsupported.
7. Do not store cookies longer than necessary.
8. Keep the product language limited to accessible media and authorized offline playback.


### Implementation Blueprint for OnlyFans

A robust OnlyFans downloader should be a detector and classifier before it is a downloader. The browser extension path should collect facts in this order:

1. Confirm that the active tab host matches `onlyfans.com` or an expected first-party subdomain.
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

### The Profile Opens but the Video Is Locked

The account may not have the subscription, PPV purchase, or message unlock required for that media.

### The Network Panel Shows `blob:`

Look for manifest and segment requests. The blob URL is not the original downloadable URL.

### The Manifest URL Stops Working

The signed URL likely expired or was tied to the prior app state. Reopen the content and detect a fresh URL.

### yt-dlp Does Not Recognize the Page

No dedicated local OnlyFans extractor was found. Use authorized direct media URLs only when visible in the browser session.

### FFmpeg Returns 403

The request is missing cookies, referer, origin, user-agent, or the signed query string has expired.

### DRM Is Present

Report unsupported. Do not attempt to bypass DRM or license checks.


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
- [FFmpeg main documentation](https://ffmpeg.org/ffmpeg.html)
- [FFmpeg protocol documentation for HTTP headers, user-agent, and referer](https://ffmpeg.org/ffmpeg-protocols.html)
- [ffprobe documentation](https://ffmpeg.org/ffprobe.html)
- [RFC 8216: HTTP Live Streaming](https://www.rfc-editor.org/rfc/rfc8216)
- [Chrome DevTools Network features reference](https://developer.chrome.com/docs/devtools/network/reference/)
- [jq manual](https://jqlang.org/manual/)
- [MDN Media Source API](https://developer.mozilla.org/en-US/docs/Web/API/Media_Source_Extensions_API)
- [MDN Encrypted Media Extensions API](https://developer.mozilla.org/en-US/docs/Web/API/Encrypted_Media_Extensions_API)
- Local tool verification: `yt-dlp 2025.12.08`, `ffmpeg`, `ffprobe`, and `jq` were present in the local environment; extractor names were checked locally without opening OnlyFans media pages.
