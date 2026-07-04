---
title: "How to Download Circle.so Videos for Free (HLS m3u8 Streams) - using yt-dlp"
description: "Step-by-step guide to download Circle.so HLS m3u8 videos with yt-dlp, auto-naming, ffmpeg optimization, and troubleshooting tips."
slug: "how-to-download-circle-so-videos-free"
source_video: "https://www.youtube.com/watch?v=ZalcLjfqDzQ"
date: "2025-11-07"
author: "howtodownloadvideosagent"
image: "https://raw.githubusercontent.com/devinschumacher/uploads/refs/heads/main/images/how-to-download-circle-so-videos-for-free-hls-m3u8-streams.jpg"
tags:
  - "guides"
  - "circle"
  - "hls"
  - "m3u8"
  - "yt-dlp"
  - "video downloads"
---

# How to Download Circle.so Videos for Free (HLS m3u8 Streams) - using yt-dlp

Circle.so hosts course videos using **HLS streaming**, where video data is split into `.ts` segments and controlled by a `.m3u8` playlist file.

This guide shows you how to **download**, **optimize**, and **auto-name** Circle Academy videos with a single command on macOS.

[![how to download circle so videos for free hls m3u8 streams](https://raw.githubusercontent.com/devinschumacher/uploads/refs/heads/main/images/how-to-download-circle-so-videos-for-free-hls-m3u8-streams.jpg)](https://www.youtube.com/watch?v=ZalcLjfqDzQ)


## 💡 **Tip:** Don’t want to do this manually?
> Try the [**Circle Video Downloader**](https://serp.ly/circle-downloader?via=github&utm_source=github&utm_medium=piggyback&utm_campaign=howtodownloadvideosagent) — it automatically detects and saves circle.so videos in one click. Or check out the code [repository](https://github.com/serpapps/circle-downloader/)


---


## 🔍 Step 1 — Identify the Stream Type

1. Open **Developer Tools → Network** in your browser.
2. Filter by **`m3u8`**.
3. You’ll see URLs like:

```
https://cdn-media.circle.so/.../playlist.m3u8
```
or
```
https://cdn-media.circle.so/.../playlist_1.m3u8
```

✅ Use the **`playlist.m3u8`** file (the master manifest).

---

## ⚙️ Step 2 — Copy the Universal Command Template

This is your reusable one-liner template.
It handles download, metadata optimization, and QuickLook thumbnail compatibility.

```bash
yt-dlp -o "%(title)s.%(ext)s" "<PASTE_M3U8_URL_HERE>"   --exec 'ffmpeg -y -i "{}" -movflags +faststart -pix_fmt yuv420p -c copy "{}_fixed.mp4" && mv "{}_fixed.mp4" "{}"'
```

💡 Keep this saved somewhere — you’ll just swap out the placeholder each time.

---

## 🚀 Step 3 — Replace the Placeholder URL and Run It

1. Go back to DevTools and copy your actual `.m3u8` link.
2. Replace `<PASTE_M3U8_URL_HERE>` in the command with your real link.
3. Paste it into **Terminal** and hit **Enter**.

✅ It will automatically:

* Download all `.ts` segments and merge them.
* Dynamically name the file based on video metadata.
* Add `faststart` and pixel formatting for Finder previews.
* Replace the original file with the optimized one.

---

## 🧠 Step 4 — What Each Flag Does

| Flag | Purpose |
|------|----------|
| `-o "%(title)s.%(ext)s"` | Dynamically name the file based on metadata. |
| `--exec 'ffmpeg ...'` | Run ffmpeg automatically after download. |
| `-movflags +faststart` | Moves metadata to the start of file for instant playback. |
| `-pix_fmt yuv420p` | Ensures Finder and QuickLook compatibility. |
| `-c copy` | No re-encoding (lossless). |

---

## 🧩 Troubleshooting

| Issue | Cause | Fix |
|-------|--------|-----|
| **Token expired** | Circle’s CDN tokens expire quickly. | Refresh the page and grab a fresh `.m3u8` link. |
| **No title metadata** | Some Circle embeds omit titles. | yt-dlp will fallback to a timestamp-based filename. |
| **No thumbnail still** | macOS caching delay. | Run `qlmanage -r cache` to refresh Finder previews. |

---

## ✅ Summary

* Find the `.m3u8` link under **Network → Media**.
* Use the master `playlist.m3u8`, not the `_1.m3u8` variant.
* Copy the **template command** from Step 2.
* Replace the placeholder URL in Step 3.
* You’ll get a Finder-previewable `.mp4` auto-named and ready to use.

---

> 💡 **Tip:** Prefer a one-click solution?
> Use the [**Circle Video Downloader**](https://serp.ly/circle-downloader?via=github&utm_source=github&utm_medium=piggyback&utm_campaign=howtodownloadvideosagent) to automatically detect, download, and merge Circle.so course videos without touching the command line.
