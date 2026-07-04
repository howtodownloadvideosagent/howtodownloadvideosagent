---
title: "How to Download Coursera Videos for Free (yt-dlp tutorial)"
description: "Download Coursera lectures with yt-dlp by capturing MP4 or HLS/DASH URLs from DevTools. Step-by-step workflow and examples."
slug: "how-to-download-coursera-videos"
source_video: "https://www.youtube.com/watch?v=IeMzUKVAcBc"
date: "2025-11-07"
author: "howtodownloadvideosagent"
image: "https://i.ytimg.com/vi/IeMzUKVAcBc/maxresdefault.jpg"
tags:
  - "guides"
  - "coursera"
  - "yt-dlp"
  - "video downloads"
---

# How to Download Coursera Videos for FREE (yt-dlp tutorial)

## Follow along with the video 👇

[![how to download coursera videos for free yt dlp tutorial](https://raw.githubusercontent.com/devinschumacher/uploads/refs/heads/main/images/how-to-download-coursera-videos-for-free-yt-dlp-tutorial.jpg)](https://www.youtube.com/watch?v=IeMzUKVAcBc)


## Steps

1. Visit the Coursera lesson page & open devtools
2. Select the .mp4
3. Copy the URL & use yt-dlp to download


## Step 1: Visit the Coursera lesson page & open devtools

- Visit the Coursera lesson page (where the video is)
- Open devtools to the network tab (right click > inspect > network) & enable "preserve logs"

## Step 2: Select the .mp4

- Filter for `mp4`
- Click the entry with `Content-Type: video/mp4` and copy the `Request URL`

![post1](https://gist.github.com/user-attachments/assets/ddf248c5-14fb-499b-8d00-d38246739b0c)


## Step 3: Copy the URL & use yt-dlp to download

- Download the video using `yt-dlp` in your Terminal program

```bash
# syntax
yt-dlp "REPLACE_ME_WITH_URL"
```

```bash
# example
yt-dlp "https://d3c33hcgiwev3.cloudfront.net/kZolKy_nEemnrA4AsaAhFA.processed/full/540p/index.mp4?Expires=1762646400&Signature=MvT4Thuyt8iKf1XR9hWDL6KtmexqybB1vLcT5jnLl-9mvW65Nkx4O~AteosR4~0NJsIoVD8FUPh7yu10QboI7NCc5hrGCOGJSYClht87aZeFd1PUdnsSNdYJ4mDk2M82pRRZGx5-PONTxqkCJqyz2SC6oGBMvRiv94KnEhbHTSU_&Key-Pair-Id=APKAJLTNE6QMUY6HBC5A"
```

> Note: The URL is time‑limited. If it expires (403/AccessDenied), re‑capture a fresh link.


## Related

