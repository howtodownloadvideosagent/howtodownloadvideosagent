---
title: "Download Skool Videos from .m3u8 Token Streams (HLS Guide)"
description: "Use your browser's network panel to capture Skool .m3u8?token URLs and download the video with yt-dlp."
slug: "download-skool-videos-m3u8-token"
source_video: "https://www.youtube.com/watch?v=7oRjGQ1Mi3s"
date: "2025-10-22"
author: "howtodownloadvideosagent"
tags:
  - "guides"
  - "skool"
  - "hls"
  - "m3u8"
  - "yt-dlp"
---

# Download Skool Videos from .m3u8 Token Streams (HLS Guide)

1. Get the `.m3u8?token=` URL from the network requests
2. Replace the `"URL"` in the prepared command with the `m3u8?token` URL


## Get the `.m3u8?token=` URL from the network requests

1. Open dev tools > network requests
2. Play the video to put the stream into the network requests
3. Filter by `m3u8` (to make your life easier)
4. Click on the m3u8 stream with the `?token` in the URL
5. Copy the URL from the Headers > Request URL

A link like this: 
```
https://stream.video.skool.com/MXf2v021K3zabLTz3wxh2dvAnRgv1lMD01LK1cqr2aG2E.m3u8?token=eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJ2IiwiZXhwIjoxNzU4NTYyOTk2LCJraWQiOiJPVjIwMHZ6SWZuZFVCNHdXdTAxbDRjb0hrYTVQQUd3TlYwMEtZSkJrQkppVlFrIiwicGxheWJhY2tfcmVzdHJpY3Rpb25faWQiOiIzMDJ2dXNuVG1lbW1QSzllOUpMaWxaUmpnVkJ3T2hTNlVLdGkyWnhJS2V1WSIsInN1YiI6Ik1YZjJ2MDIxSzN6YWJMVHozd3hoMmR2QW5SZ3YxbE1EMDFMSzFjcXIyYUcyRSJ9.1R6_xDCcKYUyUWUISfiyjKnd0Xm8laMDRKSpCd0lkziGnm95CLqPZlW4Bsj5T3xB9v2NgFQ-Ql0N0HewS685-l6r3WV8g5ddbEWQDWpmyHVDNq_Vc1PD9w-e9CIkXruzCp6-kREQtrrGgCmx2p_V5aLOjCLeVNOgRwOFuFHZxXAXlLJt-4vXqEgfRyzDVRrze5sskn1Xm5GeUQPtNWWnDWRx6k3FGQdz8BRIhucJ771TLBpTTnNgPPRINzmZeBCuVY1ZAJt84EDLKX2XCuo8MUu4vRz6P_6tLeWoljy2JlEaUD-8UbOQRYIlUVj2lYrP4iZTon7x70HvLm0SqmT8OA&CMCD=cid%3D%22MXf2v021K3zabLTz3wxh2dvAnRgv1lMD01LK1cqr2aG2E%22%2Csid%3D%2262f1bce3-68bd-46ae-a45d-2f746c6cb24b%22
```

## 2. Replace the `"URL"` in the prepared command with the `m3u8?token` URL

command
```bash
yt-dlp --referer "https://skool.com" "URL" -o "output.mp4"
```

It should become something like this:
```bash
yt-dlp --referer "https://skool.com" "https://stream.video.skool.com/MXf2v021K3zabLTz3wxh2dvAnRgv1lMD01LK1cqr2aG2E.m3u8?token=eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJ2IiwiZXhwIjoxNzU4NTYyOTk2LCJraWQiOiJPVjIwMHZ6SWZuZFVCNHdXdTAxbDRjb0hrYTVQQUd3TlYwMEtZSkJrQkppVlFrIiwicGxheWJhY2tfcmVzdHJpY3Rpb25faWQiOiIzMDJ2dXNuVG1lbW1QSzllOUpMaWxaUmpnVkJ3T2hTNlVLdGkyWnhJS2V1WSIsInN1YiI6Ik1YZjJ2MDIxSzN6YWJMVHozd3hoMmR2QW5SZ3YxbE1EMDFMSzFjcXIyYUcyRSJ9.1R6_xDCcKYUyUWUISfiyjKnd0Xm8laMDRKSpCd0lkziGnm95CLqPZlW4Bsj5T3xB9v2NgFQ-Ql0N0HewS685-l6r3WV8g5ddbEWQDWpmyHVDNq_Vc1PD9w-e9CIkXruzCp6-kREQtrrGgCmx2p_V5aLOjCLeVNOgRwOFuFHZxXAXlLJt-4vXqEgfRyzDVRrze5sskn1Xm5GeUQPtNWWnDWRx6k3FGQdz8BRIhucJ771TLBpTTnNgPPRINzmZeBCuVY1ZAJt84EDLKX2XCuo8MUu4vRz6P_6tLeWoljy2JlEaUD-8UbOQRYIlUVj2lYrP4iZTon7x70HvLm0SqmT8OA&CMCD=cid%3D%22MXf2v021K3zabLTz3wxh2dvAnRgv1lMD01LK1cqr2aG2E%22%2Csid%3D%2262f1bce3-68bd-46ae-a45d-2f746c6cb24b%22" -o "output.mp4"
```

## Bonus! If you want to download larger streams faster...

You can add ` --concurrent-fragments NUM` to the end of the command (where "NUM" should be replaced by a number, like 5,10,15,etc.

```bash
yt-dlp --referer "https://skool.com" "URL" -o "output.mp4" --concurrent-fragments 10
```
