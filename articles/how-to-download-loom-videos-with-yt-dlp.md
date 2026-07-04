---
title: "How to Download Loom Videos: A Complete Step-by-Step Guide (With Real Examples & Command Cheatsheet)"
description: "Step-by-step Loom download guide with yt-dlp covering share links, embeds, separate streams, and ready-to-copy command cheat sheets."
slug: "how-to-download-loom-videos-with-yt-dlp"
source_video: "https://www.youtube.com/watch?v=L45HqehsV6M"
date: "2025-10-25"
author: "howtodownloadvideosagent"
tags:
  - "guides"
  - "loom"
  - "yt-dlp"
  - "video downloads"
---

# How to Download Loom Videos: A Complete Step-by-Step Guide (With Real Examples & Command Cheatsheet)

Loom is a great tool for recording and sharing videos, but the platform has restrictions when it comes to downloading — especially if you’re not on a paid plan anymore.

You might have dozens or even hundreds of old recordings and realize you don’t have a way to save them locally (i did)

That’s frustrating, but it’s fixable. In this article, I’ll walk you through the exact process of detecting, inspecting, and downloading Loom videos using **yt-dlp**.

We’ll cover several different real-world scenarios you’ll encounter — from normal share links, to embeds, to videos with separate audio and video streams.


👉 Want the easy way?

Skip the terminal and use the [Loom Video Downloader](https://serp.ly/loom-downloader?via=github&utm_source=github&utm_medium=piggyback&utm_campaign=howtodownloadvideosagent) — a browser-based tool that lets you grab your Loom videos in just a couple of clicks.


Or check out the official repository: https://github.com/serpapps/loom-video-downloader

---

## Prerequisites

Before we dive in, let’s make sure you have the tools set up:

- **yt-dlp**: The open-source command-line downloader we’ll use.
  - macOS:
    ```bash
    brew install yt-dlp
    ```
  - Windows/Linux: download the binary from [yt-dlp GitHub releases](https://github.com/yt-dlp/yt-dlp).

- **Browser DevTools**: You’ll need to inspect the page source in Chrome, Firefox, or Safari to grab the actual Loom media URL.
  - Right-click → **Inspect** → use the **Elements** tab.
  - Search for `loom.com` to locate the video’s URL.

Once that’s set up, you’re ready to follow along.

---

### Example 1: M3U8 Raw URL With MP4 Transcoded

This is the most common case: a Loom share page where the video is delivered as a raw `.m3u8` stream but also has an MP4 transcoded version available.

**Step 1 – Detect the URL**
- Open DevTools and search `loom.com`.
- Copy the `share` link (e.g. `https://www.loom.com/share/43d05f362f734614a2e81b4694a3a523`).

**Step 2 – Inspect formats**
```bash
yt-dlp -F "https://www.loom.com/share/43d05f362f734614a2e81b4694a3a523"
````

You’ll see output showing `http-transcoded mp4` alongside `hls-raw`.

**Step 3 – Download**

```bash
yt-dlp "https://www.loom.com/share/43d05f362f734614a2e81b4694a3a523"
```

---

### Example 2: WebM Raw by Default

Sometimes yt-dlp defaults to a `.webm` download (from `http-raw`), even though an MP4 version exists.

**Step 1 – Detect URL**
Grab it from DevTools:
`https://www.loom.com/share/c43a642f815f4378b6f80a889bb73d8d`

**Step 2 – Run a basic download**

```bash
yt-dlp "https://www.loom.com/share/c43a642f815f4378b6f80a889bb73d8d"
```

**Result:** You’ll get a `.webm` file.

---

### Example 3: Forcing MP4 Instead of WebM

If you prefer MP4, specify the format explicitly.

**Step 1 – List formats**

```bash
yt-dlp -F "https://www.loom.com/share/c43a642f815f4378b6f80a889bb73d8d"
```

**Step 2 – Choose MP4**

```bash
yt-dlp -f http-transcoded "https://www.loom.com/share/c43a642f815f4378b6f80a889bb73d8d"
```

**Result:** This downloads the MP4 version instead of WebM.

---

### Example 4: M3U8 With Subtitles

Some Loom videos include captions in multiple formats.

**Step 1 – Check for subtitles**

```bash
yt-dlp --list-subs "https://www.loom.com/share/9458bcbf79784162aa62ffb8dd66201b"
```

Output shows `en.vtt`.

**Step 2 – Download with subs**

```bash
yt-dlp --write-subs --sub-format vtt "https://www.loom.com/share/9458bcbf79784162aa62ffb8dd66201b"
```

**Result:** You’ll get both the MP4 video and a `.vtt` subtitle file.

---

### Example 5: MPD Raw URL

Occasionally Loom delivers via an MPD manifest.

**Step 1 – Grab the URL**
`https://www.loom.com/share/24351eb8b317420289b158e4b7e96ff2`

**Step 2 – Download**

```bash
yt-dlp "https://www.loom.com/share/24351eb8b317420289b158e4b7e96ff2"
```

**Notes:** This video also has no audio.

---

### Example 6: Embed With Split Audio & Video

Some embeds separate the audio and video tracks — you’ll need to merge them.

**Step 1 – Detect embed URL**
`https://www.loom.com/embed/ddcf1c1ad21f451ea7468b1e33917e4e`

**Step 2 – Inspect formats**
Lists `hls-raw-audio (audio only)` and `hls-raw-1500 (video only)`.

**Step 3 – Download & merge**

```bash
yt-dlp -f "bestvideo+bestaudio" "https://www.loom.com/embed/ddcf1c1ad21f451ea7468b1e33917e4e"
```

**Notes:** Always use the `+` operator to join separate streams.

💡 Tip: If dealing with **embeds, subtitles, or audio/video splits feels too complicated**, the [Loom Video Downloader](https://serp.ly/loom-downloader?via=github&utm_source=github&utm_medium=piggyback&utm_campaign=howtodownloadvideosagent) automates the entire process — no commands, no DevTools, just copy-paste your Loom link and download.

---

### Example 7: Community Page With Multiple Qualities

On Loom community pages, you’ll often see multiple resolutions.

**Step 1 – Grab the embed URL**
`https://www.loom.com/embed/2a742981490b4c649ce429d75f70fd73`

**Step 2 – List formats**

```bash
yt-dlp -F "https://www.loom.com/embed/2a742981490b4c649ce429d75f70fd73"
```

Shows:

* `hls-raw-1500` → 720p
* `hls-raw-3200` → 1080p

**Step 3 – Download default (720p)**

```bash
yt-dlp "https://www.loom.com/embed/2a742981490b4c649ce429d75f70fd73"
```

**Step 4 – Download 1080p specifically**

```bash
yt-dlp -f hls-raw-3200 "https://www.loom.com/embed/2a742981490b4c649ce429d75f70fd73"
```

---

## Conclusion

That’s seven concrete examples of downloading Loom videos under different conditions. The workflow is always the same:

1. **Detect** the Loom URL with DevTools.
2. **Inspect** formats with `yt-dlp -F`.
3. **Extract/download** with the right flags.

By practicing with these examples, you’ll be ready for nearly any variation Loom throws at you — whether it’s MP4 vs WebM, subtitles, split streams, or multiple resolutions.

⚡ Want to save time and skip the terminal? Use the [Loom Video Downloader](https://serp.ly/loom-downloader?via=github&utm_source=github&utm_medium=piggyback&utm_campaign=howtodownloadvideosagent) for one-click downloads of your Loom recordings.


👉 For the full **command reference and raw steps**, check out this companion gist:
[How To Download Loom Videos For Free! 🚀 DL Without Subscription | ~10 Examples (.mp4, HLS, .m3u8) — STEP BY STEP](https://gist.github.com/devinschumacher/b7be00df9d9809d0ea55663d88dc9d3c)


## Download Loom Videos yt-dlp Command Cheatsheet

Here’s the full article again, now with a **compact command cheat sheet section at the bottom** in Markdown so you can copy/paste directly:


# Click the image below to watch the video 👇

[![How to Download Loom Videos tutorial thumbnail](https://raw.githubusercontent.com/devinschumacher/uploads/refs/heads/main/images/how-to-download-loom-videos-for-free-dl-without-subscription-10-examples-mp4-hls-m3u8-1757463216973.jpg)](https://www.youtube.com/watch?v=L45HqehsV6M)

# How to Download Loom Videos: A Complete Step-by-Step Guide (With Real Examples)

Loom is a great tool for recording and sharing videos, but the platform has restrictions when it comes to downloading — especially if you’re not on a paid plan anymore. You might have dozens or even hundreds of old recordings and realize you don’t have a way to save them locally.

That’s frustrating, but it’s fixable. In this article, I’ll walk you through the exact process of detecting, inspecting, and downloading Loom videos using **yt-dlp**. We’ll cover several different real-world scenarios you’ll encounter — from normal share links, to embeds, to videos with separate audio and video streams.

👉 Want the easy way? Skip the terminal and use the [Loom Video Downloader](https://serp.ly/loom-downloader?via=github&utm_source=github&utm_medium=piggyback&utm_campaign=howtodownloadvideosagent) — a browser-based tool that lets you grab your Loom videos in just a couple of clicks.

👉 For the full **command reference and raw steps**, check out this companion gist:
[How To Download Loom Videos For Free! 🚀 DL Without Subscription | ~10 Examples (.mp4, HLS, .m3u8) — STEP BY STEP](https://gist.github.com/devinschumacher/b7be00df9d9809d0ea55663d88dc9d3c)

---

## Prerequisites

Before we dive in, let’s make sure you have the tools set up:

- **yt-dlp**: The open-source command-line downloader we’ll use.
  - macOS:
    ```bash
    brew install yt-dlp
    ```
  - Windows/Linux: download the binary from [yt-dlp GitHub releases](https://github.com/yt-dlp/yt-dlp).

- **Browser DevTools**: You’ll need to inspect the page source in Chrome, Firefox, or Safari to grab the actual Loom media URL.
  - Right-click → **Inspect** → use the **Elements** tab.
  - Search for `loom.com` to locate the video’s URL.

Once that’s set up, you’re ready to follow along.

---

### Example 1: M3U8 Raw URL With MP4 Transcoded

```bash
yt-dlp -F "https://www.loom.com/share/43d05f362f734614a2e81b4694a3a523"
yt-dlp "https://www.loom.com/share/43d05f362f734614a2e81b4694a3a523"
````

*Note: this video has no sound.*

---

### Example 2: WebM Raw by Default

```bash
yt-dlp "https://www.loom.com/share/c43a642f815f4378b6f80a889bb73d8d"
```

*Result: downloads `.webm` by default.*

---

### Example 3: Forcing MP4 Instead of WebM

```bash
yt-dlp -F "https://www.loom.com/share/c43a642f815f4378b6f80a889bb73d8d"
yt-dlp -f http-transcoded "https://www.loom.com/share/c43a642f815f4378b6f80a889bb73d8d"
```

---

### Example 4: M3U8 With Subtitles

```bash
yt-dlp --list-subs "https://www.loom.com/share/9458bcbf79784162aa62ffb8dd66201b"
yt-dlp --write-subs --sub-format vtt "https://www.loom.com/share/9458bcbf79784162aa62ffb8dd66201b"
```

---

### Example 5: MPD Raw URL

```bash
yt-dlp "https://www.loom.com/share/24351eb8b317420289b158e4b7e96ff2"
```

*Note: no audio.*

---

### Example 6: Embed With Split Audio & Video

```bash
yt-dlp -f "bestvideo+bestaudio" "https://www.loom.com/embed/ddcf1c1ad21f451ea7468b1e33917e4e"
```

💡 Tip: If dealing with **embeds, subtitles, or audio/video splits feels too complicated**, the [Loom Video Downloader](https://serp.ly/loom-downloader?via=github&utm_source=github&utm_medium=piggyback&utm_campaign=howtodownloadvideosagent) automates the entire process — no commands, no DevTools, just copy-paste your Loom link and download.

---

### Example 7: Community Page With Multiple Qualities

```bash
yt-dlp -F "https://www.loom.com/embed/2a742981490b4c649ce429d75f70fd73"
yt-dlp "https://www.loom.com/embed/2a742981490b4c649ce429d75f70fd73"
yt-dlp -f hls-raw-3200 "https://www.loom.com/embed/2a742981490b4c649ce429d75f70fd73"
```

---

## Conclusion

That’s seven concrete examples of downloading Loom videos under different conditions. The workflow is always the same:

1. **Detect** the Loom URL with DevTools.
2. **Inspect** formats with `yt-dlp -F`.
3. **Extract/download** with the right flags.

By practicing with these examples, you’ll be ready for nearly any variation Loom throws at you — whether it’s MP4 vs WebM, subtitles, split streams, or multiple resolutions.

⚡ Want to save time and skip the terminal? Use the [Loom Video Downloader](https://serp.ly/loom-downloader?via=github&utm_source=github&utm_medium=piggyback&utm_campaign=howtodownloadvideosagent) for one-click downloads of your Loom recordings.

---

## Quick Command Cheat Sheet

| Example | Scenario                   | Command(s)                                                                                           |
| ------- | -------------------------- | ---------------------------------------------------------------------------------------------------- |
| 1       | M3U8 raw w/ MP4 transcoded | `yt-dlp "https://www.loom.com/share/43d05f362f734614a2e81b4694a3a523"`                               |
| 2       | WebM raw default           | `yt-dlp "https://www.loom.com/share/c43a642f815f4378b6f80a889bb73d8d"`                               |
| 3       | Force MP4 instead of WebM  | `yt-dlp -f http-transcoded "https://www.loom.com/share/c43a642f815f4378b6f80a889bb73d8d"`            |
| 4       | With subtitles             | `yt-dlp --write-subs --sub-format vtt "https://www.loom.com/share/9458bcbf79784162aa62ffb8dd66201b"` |
| 5       | MPD raw                    | `yt-dlp "https://www.loom.com/share/24351eb8b317420289b158e4b7e96ff2"`                               |
| 6       | Split audio/video          | `yt-dlp -f "bestvideo+bestaudio" "https://www.loom.com/embed/ddcf1c1ad21f451ea7468b1e33917e4e"`      |
| 7       | Community page (1080p)     | `yt-dlp -f hls-raw-3200 "https://www.loom.com/embed/2a742981490b4c649ce429d75f70fd73"`               |

### Command flags

```
yt-dlp get file format info yt-dlp -F
yt-dlp get subtitles info yt-dlp --list-subs
yt-dlp get thumbnail info yt-dlp --list-thumbnails
```
