---
title: "How to Download HLS/M3U8 Streaming Videos"
description: "File Extensions: .m3u8, .m3u MIME Types: application/x-mpegURL, application/vnd.apple.mpegurl, audio/x-mpegurl Container: Playlist format (references video/audio segments) Streaming: Adaptive bitrate streaming protocol"
slug: "how-to-download-hls-m3u8-streaming-videos"
source_video: "https://www.youtube.com/watch?v=7oRjGQ1Mi3s"
date: "2025-10-22"
author: "howtodownloadvideosagent"
tags:
  - "guides"
  - "hls"
  - "m3u8"
  - "video downloads"
---

# How to Download HLS/M3U8 Streaming Videos

**File Extensions**: `.m3u8`, `.m3u`
**MIME Types**: `application/x-mpegURL`, `application/vnd.apple.mpegurl`, `audio/x-mpegurl`
**Container**: Playlist format (references video/audio segments)
**Streaming**: Adaptive bitrate streaming protocol

## Overview

HTTP Live Streaming (HLS) is Apple's adaptive bitrate streaming protocol that delivers video and audio content over HTTP. It works by breaking the stream into small HTTP-based segments and creating a playlist file (M3U8) that references these segments. Many platforms use HLS alongside other protocols like [RTMP streaming](https://apps.serp.co/blog/how-to-download-rtmp-live-streams), [DASH protocols](https://apps.serp.co/blog/how-to-download-dash-streaming-videos), and [WebRTC](https://apps.serp.co/blog/how-to-download-webrtc-video-streams) for different use cases.

## HLS Structure

### Master Playlist (m3u8)
Contains references to variant streams with different bitrates/resolutions:

```
#EXTM3U
#EXT-X-VERSION:6
#EXT-X-STREAM-INF:BANDWIDTH=1280000,RESOLUTION=720x404,CODECS="avc1.77.30,mp4a.40.2"
variant1/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=2560000,RESOLUTION=1280x720,CODECS="avc1.77.31,mp4a.40.2"
variant2/playlist.m3u8
```

### Variant Playlist (m3u8)
Contains individual segment references:

```
#EXTM3U
#EXT-X-VERSION:6
#EXT-X-TARGETDURATION:10
#EXT-X-MEDIA-SEQUENCE:0
#EXTINF:10.0,
segment000.ts
#EXTINF:10.0,
segment001.ts
```

## Detection Methods

### 1. Network Traffic Analysis

#### Browser Developer Tools
```javascript
// Open Network tab and filter by:
// - Type: Media or XHR
// - Name contains: .m3u8
// Look for playlist requests during video playback
```

#### JavaScript Network Interception
```javascript
// Intercept fetch requests for M3U8
const originalFetch = window.fetch;
window.fetch = function(...args) {
  const url = args[0];
  if (typeof url === 'string' && url.includes('.m3u8')) {
    console.log('HLS Playlist detected:', url);
  }
  return originalFetch.apply(this, args);
};
```

### 2. DOM Analysis

#### HTML Video Element Sources
```javascript
// Check video elements for HLS sources
document.querySelectorAll('video').forEach(video => {
  const src = video.src || video.currentSrc;
  if (src && src.includes('.m3u8')) {
    console.log('HLS source found:', src);
  }

  // Check source children
  video.querySelectorAll('source').forEach(source => {
    if (source.src && source.src.includes('.m3u8')) {
      console.log('HLS source found:', source.src);
    }
  });
});
```

#### Video.js Player Detection
```javascript
// Video.js specific detection
if (window.videojs) {
  const players = videojs.getPlayers();
  Object.values(players).forEach(player => {
    const tech = player.tech();
    if (tech && tech.hls) {
      console.log('HLS URL:', tech.hls.playlists.master.uri);
    }
  });
}
```

### 3. Custom Player Framework Detection

#### JW Player
```javascript
// JW Player HLS detection
if (window.jwplayer) {
  jwplayer().on('ready', function() {
    const playlist = this.getPlaylist();
    playlist.forEach(item => {
      if (item.file && item.file.includes('.m3u8')) {
        console.log('JW Player HLS:', item.file);
      }
    });
  });
}
```

#### Hls.js Library Detection
```javascript
// Hls.js detection
if (window.Hls && window.Hls.isSupported()) {
  const originalLoadSource = window.Hls.prototype.loadSource;
  window.Hls.prototype.loadSource = function(url) {
    console.log('Hls.js loading:', url);
    return originalLoadSource.call(this, url);
  };
}
```

## Injection and Extraction Methods

### 1. Browser Extension Content Script
```javascript
// Content script for browser extension
function detectHLSStreams() {
  const streams = new Set();

  // Network request monitoring
  chrome.webRequest.onBeforeRequest.addListener(
    function(details) {
      if (details.url.includes('.m3u8')) {
        streams.add(details.url);
        console.log('HLS Stream detected:', details.url);
      }
    },
    {urls: ["<all_urls>"]},
    ["requestBody"]
  );

  return Array.from(streams);
}
```

### 2. Tampermonkey/Greasemonkey Script
```javascript
// ==UserScript==
// @name         HLS Stream Detector
// @namespace    http://tampermonkey.net/
// @version      0.1
// @description  Detect HLS streams on any website
// @match        *://*/*
// @grant        none
// ==/UserScript==

(function() {
    'use strict';

    const detectedStreams = new Set();

    // Override XMLHttpRequest
    const originalOpen = XMLHttpRequest.prototype.open;
    XMLHttpRequest.prototype.open = function(method, url) {
        if (url.includes('.m3u8')) {
            detectedStreams.add(url);
            console.log('HLS detected via XHR:', url);
        }
        return originalOpen.apply(this, arguments);
    };

    // Add UI indicator
    const indicator = document.createElement('div');
    indicator.style.position = 'fixed';
    indicator.style.top = '10px';
    indicator.style.right = '10px';
    indicator.style.zIndex = '10000';
    indicator.style.padding = '10px';
    indicator.style.background = 'red';
    indicator.style.color = 'white';
    indicator.textContent = 'HLS Detector Active';
    document.body.appendChild(indicator);
})();
```

### 3. Puppeteer/Selenium Automation
```javascript
// Puppeteer HLS detection
const puppeteer = require('puppeteer');

async function detectHLS(url) {
  const browser = await puppeteer.launch();
  const page = await browser.newPage();

  const m3u8URLs = [];

  // Listen for network requests
  await page.setRequestInterception(true);
  page.on('request', request => {
    if (request.url().includes('.m3u8')) {
      m3u8URLs.push(request.url());
    }
    request.continue();
  });

  await page.goto(url);

  // Wait for video to start loading
  await page.waitForTimeout(5000);

  await browser.close();
  return m3u8URLs;
}
```

## Download Methods

### 1. yt-dlp (Recommended)
```bash
# Basic download
yt-dlp "https://example.com/playlist.m3u8"

# With specific format selection
yt-dlp -f "best[height<=720]" "https://example.com/playlist.m3u8"

# With headers (important for protected streams)
yt-dlp --add-header "Referer: https://example.com/" \
       --add-header "Origin: https://example.com/" \
       "https://example.com/playlist.m3u8"

# With cookies
yt-dlp --cookies cookies.txt "https://example.com/playlist.m3u8"

# Concurrent fragments for faster download
yt-dlp --concurrent-fragments 5 "https://example.com/playlist.m3u8"
```

### 2. ffmpeg Direct Download
```bash
# Basic ffmpeg download
ffmpeg -i "https://example.com/playlist.m3u8" -c copy output.mp4

# With headers
ffmpeg -headers "Referer: https://example.com/" \
       -i "https://example.com/playlist.m3u8" \
       -c copy output.mp4

# With specific codec selection
ffmpeg -i "https://example.com/playlist.m3u8" \
       -c:v h264 -c:a aac \
       -preset fast output.mp4
```

### 3. streamlink
```bash
# Basic streamlink usage
streamlink "https://example.com/playlist.m3u8" best

# Output to file
streamlink -o output.mp4 "https://example.com/playlist.m3u8" best

# With headers
streamlink --http-header "Referer=https://example.com/" \
           "https://example.com/playlist.m3u8" best
```

### 4. Node.js Implementation
```javascript
const fs = require('fs');
const https = require('https');
const path = require('path');

async function downloadHLS(m3u8Url, outputDir) {
  // Download master playlist
  const masterPlaylist = await downloadFile(m3u8Url);

  // Parse for variant streams
  const variants = parseM3U8(masterPlaylist);

  // Select best quality
  const bestVariant = variants.sort((a, b) => b.bandwidth - a.bandwidth)[0];

  // Download variant playlist
  const variantPlaylist = await downloadFile(bestVariant.url);

  // Parse segments
  const segments = parseSegments(variantPlaylist);

  // Download all segments
  const segmentFiles = [];
  for (const segment of segments) {
    const segmentPath = path.join(outputDir, segment.filename);
    await downloadFile(segment.url, segmentPath);
    segmentFiles.push(segmentPath);
  }

  // Concatenate segments (requires ffmpeg)
  return concatenateSegments(segmentFiles, path.join(outputDir, 'output.mp4'));
}

function parseM3U8(content) {
  const lines = content.split('\n');
  const variants = [];

  for (let i = 0; i < lines.length; i++) {
    if (lines[i].startsWith('#EXT-X-STREAM-INF:')) {
      const bandwidth = lines[i].match(/BANDWIDTH=(\d+)/)?.[1];
      const resolution = lines[i].match(/RESOLUTION=(\d+x\d+)/)?.[1];
      const url = lines[i + 1];

      variants.push({ bandwidth: parseInt(bandwidth), resolution, url });
    }
  }

  return variants;
}
```

## Common Issues and Solutions

### 1. CORS Issues
```javascript
// Use proxy or CORS-anywhere for browser-based requests
const proxyUrl = 'https://cors-anywhere.herokuapp.com/';
const m3u8Url = 'https://example.com/playlist.m3u8';
fetch(proxyUrl + m3u8Url);
```

### 2. Authentication Required
```bash
# Use cookies from authenticated session
yt-dlp --cookies cookies.txt "https://example.com/playlist.m3u8"

# Or use session/token headers
yt-dlp --add-header "Authorization: Bearer TOKEN" \
       "https://example.com/playlist.m3u8"
```

### 3. Geo-blocking
```bash
# Use proxy
yt-dlp --proxy http://proxy-server:port "https://example.com/playlist.m3u8"

# Use specific user agent
yt-dlp --user-agent "Mozilla/5.0..." "https://example.com/playlist.m3u8"
```

## Platform-Specific Implementations

### Platform Examples

Many platforms like [Skool](https://apps.serp.co/skool-video-downloader) use HLS with these characteristics:
- Master playlists at various CDNs (fastly, b-cdn.net)
- Variant streams with different resolutions
- Requires proper Referer and Origin headers

```typescript
// From the codebase - Skool HLS detection
function scoreSkoolM3U8(url: string, contentType: string): number {
  const u = new URL(url);
  const host = u.host.toLowerCase();
  const path = u.pathname.toLowerCase();

  if (/\.b-cdn\.net$/i.test(host) && /\/playlist\.m3u8(?:$|\?)/i.test(path)) return 5;
  if (host === 'stream.video.skool.com') return 4;
  if (/^manifest-/.test(host) && /fastly\.video\.skool\.com$/i.test(host)) return 3;

  return 1;
}
```

### Advanced Techniques

#### 1. Dynamic Playlist Refresh
```javascript
// Monitor live HLS streams that update their playlists
async function monitorLiveHLS(playlistUrl) {
  let lastSequence = -1;

  setInterval(async () => {
    const playlist = await fetch(playlistUrl).then(r => r.text());
    const currentSequence = parseMediaSequence(playlist);

    if (currentSequence > lastSequence) {
      console.log('New segments available');
      lastSequence = currentSequence;
    }
  }, 10000);
}
```

#### 2. Multi-variant Download
```bash
# Download multiple qualities
yt-dlp -f "best[height<=1080],best[height<=720],best[height<=480]" \
       --output "%(title)s_%(height)sp.%(ext)s" \
       "https://example.com/playlist.m3u8"

## Conclusion

HLS/M3U8 is a robust and widely-adopted streaming protocol that provides excellent compatibility across devices and platforms. Understanding its structure helps you download content from platforms like [Udemy](https://apps.serp.co/udemy-video-downloader), [Coursera](https://apps.serp.co/coursera-downloader), and [Teachable](https://apps.serp.co/teachable-video-downloader) that often use HLS for course delivery.
```

## Related Tools

 - Specialized tool for downloading HLS/M3U8 streams
 - Universal streaming video downloader
 - Download from YouTube which uses adaptive streaming
 - Download Vimeo videos that use HLS
 - Download Twitch streams and VODs
 - Download from Dailymotion


## YouTube Tutorial

[Watch on YouTube](https://www.youtube.com/watch?v=BKKwxqrG7Jw)
