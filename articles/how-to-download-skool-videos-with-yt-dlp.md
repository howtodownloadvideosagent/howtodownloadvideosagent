---
title: "How to Download Skool Videos with yt-dlp"
description: "Quick instructions for saving Skool HLS streams using yt-dlp with custom output, referer headers, and format tuning."
slug: "how-to-download-skool-videos-with-yt-dlp"
source_video: "https://www.youtube.com/watch?v=TebnzxkXYTM"
date: "2025-10-25"
author: "howtodownloadvideosagent"
tags:
  - "guides"
  - "skool"
  - "yt-dlp"
  - "video downloads"
---

# How to Download Skool Videos with yt-dlp

Skool delivers videos over **HLS** (`.m3u8` playlists) with **short-lived signed tokens**. These tokens expire, so you must grab a fresh link from DevTools → Network when you want to download.

---

## 1. Copy the `.m3u8` Link

From DevTools (Network tab, filter for `m3u8`), copy the full URL.
It will look like this (very long):

```
https://stream.video.skool.com/DsdzJ4y...Zz00.m3u8?token=eyJhbGciOi...
```

---

## 2. Use yt-dlp with Referer and Safe Output

If you just paste the link, yt-dlp will try to use the full token in the filename → macOS will complain (`[Errno 63] File name too long`).
Fix = supply your own short `-o` filename.

Example:

```bash
yt-dlp -N 16 \
  --referer "https://skool.com" \
  --merge-output-format mp4 \
  --remux-video mp4 \
  --postprocessor-args "ffmpeg:-movflags +faststart" \
  -o "skool_video.mp4" \
  "https://stream.video.skool.com/DsdzJ4yCk3y00EW2MEKubPX01lXnIAw001M86u5q5TzZz00.m3u8?token=eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..."
```

---

## 3. Flags Explained

* `-N 16` → downloads 16 fragments in parallel (speeds up long videos).
* `--referer "https://skool.com"` → Skool’s CDN requires this header.
* `--merge-output-format mp4 --remux-video mp4` → final MP4 output, no re-encode.
* `--postprocessor-args "ffmpeg:-movflags +faststart"` → makes the MP4 seekable immediately.
* `-o "skool_video.mp4"` → sets a short safe filename.

---

## 4. Using Metadata Titles

If you prefer automatic naming, use:

```bash
-o "%(title)s.%(ext)s"
```

So:

```bash
yt-dlp -N 16 \
  --referer "https://skool.com" \
  --merge-output-format mp4 --remux-video mp4 \
  --postprocessor-args "ffmpeg:-movflags +faststart" \
  -o "%(title)s.%(ext)s" \
  "https://stream.video.skool.com/DsdzJ4yCk3y00EW2MEKubPX01lXnIAw001M86u5q5TzZz00.m3u8?token=eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJ2IiwiZXhwIjoxNzU5MDk5NTcxLCJraWQiOiJPVjIwMHZ6SWZuZFVCNHdXdTAxbDRjb0hrYTVQQUd3TlYwMEtZSkJrQkppVlFrIiwicGxheWJhY2tfcmVzdHJpY3Rpb25faWQiOiIzMDJ2dXNuVG1lbW1QSzllOUpMaWxaUmpnVkJ3T2hTNlVLdGkyWnhJS2V1WSIsInN1YiI6IkRzZHpKNHlDazN5MDBFVzJNRUt1YlBYMDFsWG5JQXcwMDFNODZ1NXE1VHpaejAwIn0.TrTbRTKE5uOkn8KTKftog98Vch_lLQGynPZzU0eNpS9qtYmlBbJA6PwK0Yo-33FGuhFZIuVbYp3574Ki4ccDBVZKvoHRN2XcVjXi5pdXBjv6_0GgYglFGjoUHhakbiQNNLVP6WvnPb14yoDlmIY-3SmoyjHGY3dU9SBtYSSy8Ebxpe0jouei09FIGFY5zqSbxqW0v-nuYNcqIlGTjgUCU8-osYGv14JNAWqTZitcg-7XN4wX2TKNunaR7v8PS90gUt1p6qUARsSJfQl1v3qkDoMVMc77Sm9QrRsXhkofZ2OCRIyOBbBQPb9Y91LGH-fT8zMgYVh-QEk844dMCNOSVA&CMCD=cid%3D%22DsdzJ4yCk3y00EW2MEKubPX01lXnIAw001M86u5q5TzZz00%22%2Csid%3D%22a65b3c3d-1437-4b36-b707-a9a48d3fc02f%22"
```

This will name the file after the video’s title instead of a random hash.

---

## 5. Notes & Gotchas

* 🔑 **Token Expiry:** the `token=...` query string expires. If you get `403 Forbidden`, grab a new `.m3u8` link.
* 🎞 **Quality:** yt-dlp will pick the best quality listed in the `.m3u8`. You can see all available with:

  ```bash
  yt-dlp --list-formats "URL"
  ```
* 📝 **Captions:** if captions are available, add `--write-subs --embed-subs`.

# Readable version using subshells (fallback method)

```bash
VIDEO_URL='https://stream.video.skool.com/DsdzJ4yCk3y00EW2MEKubPX01lXnIAw001M86u5q5TzZz00.m3u8?token=eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJ2IiwiZXhwIjoxNzU5MTAwNTg3LCJraWQiOiJPVjIwMHZ6SWZuZFVCNHdXdTAxbDRjb0hrYTVQQUd3TlYwMEtZSkJrQkppVlFrIiwicGxheWJhY2tfcmVzdHJpY3Rpb25faWQiOiIzMDJ2dXNuVG1lbW1QSzllOUpMaWxaUmpnVkJ3T2hTNlVLdGkyWnhJS2V1WSIsInN1YiI6IkRzZHpKNHlDazN5MDBFVzJNRUt1YlBYMDFsWG5JQXcwMDFNODZ1NXE1VHpaejAwIn0.t3RDhopUU2FwgayBuZ7Y52HUMAcEOM2RcWCF6_Je5W1osyPjgHDFlSAiQOtdlXixUcQ1ZzUnN2Ww2TsSKCN6_wFhPf9RtpgzzzNyr_bNhG_sjaj4hQBQzKsWwN7A7u2a1Af0hSgWQ6_Txcso9KYQgXk5i7rAKUywP5PBqi1NZlnFI6q0EM312l6ebTfp9n-ICzCu0bEIIKfLCFY17Tcm67wEjbBxvznYpaYbOx7LKB9xjguM5BJREyFiwCZDIVKxIuVSahvnmkCErYrYidJzYkHi1Lw0oY6VigW-bPvfhygzwlx3rUxwdpLYOie_HhX26ib1ujrRf4Lwe89pPAM1nA&CMCD=cid%3D%22DsdzJ4yCk3y00EW2MEKubPX01lXnIAw001M86u5q5TzZz00%22%2Csid%3D%22892678fc-4137-4f1c-817a-55a38d4f3bab%22'


( yt-dlp -N 16 -f best --hls-prefer-native \
    --referer 'https://skool.com' \
    --add-header 'Origin: https://skool.com' \
    --add-header 'User-Agent: Mozilla/5.0' \
    --remux-video mp4 --merge-output-format mp4 \
    --postprocessor-args "ffmpeg:-movflags +faststart" \
    -o "%(title).200B.%(ext)s" "$VIDEO_URL" ) || \


( yt-dlp -N 16 -f best --hls-prefer-native \
    --referer 'https://www.skool.com' \
    --add-header 'Origin: https://www.skool.com' \
    --add-header 'User-Agent: Mozilla/5.0' \
    --remux-video mp4 --merge-output-format mp4 \
    --postprocessor-args "ffmpeg:-movflags +faststart" \
    -o "%(title).200B.%(ext)s" "$VIDEO_URL" ) || \


( yt-dlp -N 16 -f best --hls-prefer-native \
    --retries 20 --fragment-retries 100 \
    --referer 'https://skool.com' \
    --add-header 'Origin: https://skool.com' \
    --add-header 'User-Agent: Mozilla/5.0' \
    --remux-video mp4 --merge-output-format mp4 \
    --postprocessor-args "ffmpeg:-movflags +faststart" \
    -o "%(title).200B.%(ext)s" "$VIDEO_URL" ) || \


( yt-dlp -N 16 -f best \
    --downloader aria2c \
    --downloader-args "aria2c:-x 16 -s 16 -k 1M --max-tries=0 --retry-wait=5" \
    --referer 'https://skool.com' \
    --add-header 'Origin: https://skool.com' \
    --add-header 'User-Agent: Mozilla/5.0' \
    --remux-video mp4 --merge-output-format mp4 \
    --postprocessor-args "ffmpeg:-movflags +faststart" \
    -o "skool_video.mp4" "$VIDEO_URL" )
```
