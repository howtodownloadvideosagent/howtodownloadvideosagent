---
title: "How to Download Password-Protected Vimeo Videos (Direct Stream)"
description: "Walk through grabbing passworded Vimeo streams with yt-dlp by pairing browser cookies, profile IDs, and passwords for direct downloads."
slug: "how-to-download-password-protected-vimeo-videos"
source_video: "https://www.youtube.com/watch?v=Of6ND8DUewc"
date: "2025-10-25"
author: "howtodownloadvideosagent"
tags:
  - "guides"
  - "vimeo"
  - "password protected"
  - "yt-dlp"
---

# How to download password protected Vimeo videos - not embedded, stream

- Example: https://vimeo.com/1100807276


## Steps

1. visit page & enter the password
2. find your chrome profile number
3. construct the correct command

# 1. visit page & enter the password

Go to the Vimeo URL & enter the password.

# 2. find your chrome profile number

1. visit `chrome://profile-internals/`
2. expand your profile & grab the number

![image](https://gist.github.com/user-attachments/assets/b4ad85e1-8eb3-4f57-8a0e-ea451a8b6341)


# 3. construct the correct command

**You will need:**

1. `https://vimeo.com/{VIDEO_ID}`
2. chrome profile number
3. password

**Your command syntax:**

```bash
yt-dlp \
  'https://vimeo.com/{VIDEO_ID}' \
  --cookies-from-browser "chrome:Profile {NUMBER}" \
  --video-password '{PASSWORD}' \
  -N 20 \
  -S 'codec:avc,res,ext' \
  --merge-output-format mp4 \
  --remux-video mp4 \
  --postprocessor-args "ffmpeg:-movflags +faststart"
```

**et voila!**

![image](https://gist.github.com/user-attachments/assets/ad68e4ff-85e5-4586-8372-43b8b1400ff8)

**it hath downloaded**

![image](https://gist.github.com/user-attachments/assets/d57c2a6e-14ec-42e9-8cde-ea014d80cb16)
