---
title: "How to Download Skool.com Videos (and the Entire Course)"
description: "Comprehensive tutorial for mirroring entire Skool classrooms using cookies, custom scripts, and wget automation."
slug: "how-to-download-skool-videos-and-entire-course"
source_video: "https://www.youtube.com/watch?v=_NinMA9Tqis"
date: "2025-10-25"
author: "howtodownloadvideosagent"
tags:
  - "guides"
  - "skool"
  - "yt-dlp"
  - "course downloads"
---

# How to download skool.com videos (and the entire course)

## Version 1 - Comprehensive, More Difficult

> You can download the entire classroom with this, but its not beginner friendly.
> If you want the beginner version to just download 1 video at a time, scroll down to the bottom!

This is not a beginner friendly video, but if you follow along I’ll show you how to get around the download protections.

If you’re looking for a simpler solution or have any questions about this process, just leave a comment and I’ll create a follow-up video.

## Watch the video

[![How to Download Videos from Skool.com tutorial thumbnail](https://raw.githubusercontent.com/devinschumacher/uploads/refs/heads/main/images/how-to-download-videos-from-skoolcom-2025-update-step-by-step-tutorial.jpg)](https://www.youtube.com/watch?v=_NinMA9Tqis)

## Prereqs

1. [homebrew](https://brew.sh/) (free)
2. wget (free)
3. [Get cookies.txt LOCALLY](https://chromewebstore.google.com/detail/get-cookiestxt-locally/cclelndahbckbenkjhflpdbgdldlbecc)
4. [visual studio code](https://code.visualstudio.com/) (free)
5. [live server extension](https://marketplace.visualstudio.com/items?itemName=ritwickdey.LiveServer) for visual studio code (free)
6. [pants](https://serp.ly/amazon/pants?via=github&utm_source=github&utm_medium=piggyback&utm_campaign=howtodownloadvideosagent) (in order for this process to work you have to be wearing pants unfortunately)

## Steps overview

1. Put on pants 👖
2. Login to skool & navigate to the classroom
3. Save the cookies with Get cookies.txt extension (netscape format)
4. Open browser console & paste in the `.js` code, press enter
5. Create a folder on your desktop called `project`
6. Put your `cookies.txt` and `urls.txt` file in the project folder
7. Open terminal application and type `cd ~/Desktop/project`
8. Run the `wget` command to start the process
9. Remove pants 🩲 and wait for it to finish


## Detailed step by step guide

### 1. Put on pants 👖

![pants](https://gist.github.com/user-attachments/assets/9f8ad92a-2304-4042-b694-0da153c7db5c)


### 2. Login to skool & navigate to the classroom


### 3. Save the cookies with Get cookies.txt extension (netscape format)


### 4. Open browser console & paste in the `.js` code, press enter


```js
// Extract all lesson URLs from Skool classroom - paste in browser console
function extractAllLessonUrls() {
    console.log("🔍 Searching for lesson URLs on this page...");

    const urls = new Set();

    // Dynamically get base URL from current page
    const currentUrl = new URL(window.location.href);
    const baseUrl = `${currentUrl.protocol}//${currentUrl.host}${currentUrl.pathname}`;

    console.log(`📍 Auto-detected base URL: ${baseUrl}`);
    console.log(`🌐 Current full URL: ${window.location.href}`);

    // Method 1: Look for direct links with md= parameters
    document.querySelectorAll('a[href*="classroom"]').forEach(link => {
        if (link.href.includes('md=')) {
            urls.add(link.href);
            console.log("Found link:", link.href);
        }
    });

    // Method 2: Look for any links containing md= in any attribute
    document.querySelectorAll('*').forEach(el => {
        // Check onclick handlers
        const onclick = el.getAttribute('onclick');
        if (onclick && onclick.includes('md=')) {
            const mdMatch = onclick.match(/md=([a-f0-9]+)/);
            if (mdMatch) {
                const url = `${baseUrl}?md=${mdMatch[1]}`;
                urls.add(url);
                console.log("Found in onclick:", url);
            }
        }

        // Check data attributes
        ['data-href', 'data-url', 'data-link', 'data-md'].forEach(attr => {
            const value = el.getAttribute(attr);
            if (value && value.includes('md=')) {
                if (value.startsWith('http')) {
                    urls.add(value);
                } else {
                    const mdMatch = value.match(/md=([a-f0-9]+)/);
                    if (mdMatch) {
                        const url = `${baseUrl}?md=${mdMatch[1]}`;
                        urls.add(url);
                        console.log(`Found in ${attr}:`, url);
                    }
                }
            }
        });

        // Check for md= in any attribute value
        for (let attr of el.attributes) {
            if (attr.value.includes('md=') && !attr.name.startsWith('data-react')) {
                const mdMatch = attr.value.match(/md=([a-f0-9]+)/g);
                if (mdMatch) {
                    mdMatch.forEach(match => {
                        const mdValue = match.replace('md=', '');
                        const url = `${baseUrl}?md=${mdValue}`;
                        urls.add(url);
                        console.log(`Found in ${attr.name}:`, url);
                    });
                }
            }
        }
    });

    // Method 3: Look in page source/innerHTML for md= patterns
    const pageSource = document.documentElement.innerHTML;
    const mdMatches = pageSource.match(/md=([a-f0-9]{32})/g);
    if (mdMatches) {
        mdMatches.forEach(match => {
            const mdValue = match.replace('md=', '');
            const url = `${baseUrl}?md=${mdValue}`;
            urls.add(url);
        });
        console.log(`Found ${mdMatches.length} md= patterns in page source`);
    }

    // Add current page URL if it has md=
    if (window.location.href.includes('md=')) {
        urls.add(window.location.href);
        console.log("Added current page:", window.location.href);
    }

    const urlArray = Array.from(urls).sort();

    console.log(`\n✅ Found ${urlArray.length} unique lesson URLs:`);
    urlArray.forEach((url, index) => {
        console.log(`${index + 1}. ${url}`);
    });

    if (urlArray.length === 0) {
        console.log("\n❌ No lesson URLs found!");
        console.log("💡 Try running this from the main classroom navigation page");
        console.log("💡 Or manually add URLs to the list below");

        // Add current URL as fallback
        urlArray.push(window.location.href);
    }

    // Create urls.txt file and download it
    const urlText = urlArray.join('\n');
    const blob = new Blob([urlText], {type: 'text/plain'});
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = 'urls.txt';
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);

    console.log("\n📁 Downloaded urls.txt with all found URLs");
    console.log("📝 Next steps:");
    console.log("1. Check the downloaded urls.txt file");
    console.log("2. Add any missing URLs manually if needed");
    console.log("3. Use: wget --input-file=urls.txt [other options]");

    return urlArray;
}

// Run the extraction
extractAllLessonUrls();
```

### 5. Create a folder on your desktop called `project`

### 6. Put your `cookies.txt` and `urls.txt` file in the project folder

### 7. Open terminal application and navigate to your project folder


```bash
cd ~/Desktop/project
```

### 8. Run the `wget` command to start the process


```bash
#!/bin/bash

# Complete website scraping with wget
wget \
  --input-file=urls.txt \
  --page-requisites \
  --html-extension \
  --convert-links \
  --restrict-file-names=windows \
  --force-directories \
  --no-clobber \
  --directory-prefix=./skool_complete \
  --load-cookies=cookies.txt \
  --user-agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36" \
  --wait=2 \
  --span-hosts \
  --recursive \
  --level=5 \
  --no-parent
```


### 9. Remove pants 🩲 and wait for it to finish

![2](https://gist.github.com/user-attachments/assets/d9889141-4333-42d2-ac87-b9498d4ce182)


### 10. How to use it

At this point you basically have a copy of what school has on their servers but you have it on your computer so you can access it anytime anywhere and without internet. Simply open the folder in Visual Studio Code and click on one of the HTML files and open with live server.


---

# How to download a single video from skool.com (easy method)

> Here's the easier version, as request by all ye youtubers!


## Step 1: Visit the page on skool.com where the video is

## Step 2: Copy and paste this command in your dev tools console (right click > inspect > go to console tab)

```js
/**
 * Skool Loom Video Extractor - Final Version
 * Extracts video from current lesson using the md query parameter
 *
 * Usage: Paste this entire script in browser console while on a Skool classroom page
 * Then, go subscribe to @devinschumacher on YouTube
 */

(function() {
    console.log('🎥 Skool Loom Video Extractor - Final Version');

    // Get the md parameter from URL
    const urlParams = new URLSearchParams(window.location.search);
    const courseId = urlParams.get('md');

    if (!courseId) {
        console.log('❌ No "md" parameter found in URL');
        alert('This page does not have a lesson ID (md parameter). Make sure you are on a lesson page.');
        return;
    }

    console.log(`📚 Looking for course with ID: ${courseId}`);

    // Function to recursively search for course by ID
    function findCourseById(obj, targetId) {
        if (!obj || typeof obj !== 'object') return null;

        // Direct match
        if (obj.id === targetId && obj.metadata && obj.metadata.videoLink) {
            return obj;
        }

        // Check if this object has a course property with matching ID
        if (obj.course && obj.course.id === targetId && obj.course.metadata && obj.course.metadata.videoLink) {
            return obj.course;
        }

        // Recursively search all properties
        for (let key in obj) {
            if (obj.hasOwnProperty(key)) {
                const result = findCourseById(obj[key], targetId);
                if (result) return result;
            }
        }

        return null;
    }

    // Get the Next.js data
    const nextDataScript = document.getElementById('__NEXT_DATA__');
    if (!nextDataScript) {
        console.log('❌ Could not find __NEXT_DATA__ script');
        alert('Could not find page data. This script works on Skool classroom pages.');
        return;
    }

    try {
        const nextData = JSON.parse(nextDataScript.textContent);
        console.log('📋 Found page data, searching for course...');

        // Search for the course
        const course = findCourseById(nextData, courseId);

        if (!course) {
            console.log('❌ Could not find course with ID:', courseId);
            alert(`Could not find lesson with ID: ${courseId}`);
            return;
        }

        // Extract video information
        const metadata = course.metadata;
        const videoUrl = metadata.videoLink.split('?')[0]; // Clean URL without parameters
        const title = metadata.title || 'Untitled Lesson';
        const duration = metadata.videoLenMs ? Math.round(metadata.videoLenMs / 1000) : null;

        console.log('✅ Found video!');
        console.log(`📹 Title: ${title}`);
        console.log(`🔗 URL: ${videoUrl}`);
        if (duration) {
            const minutes = Math.floor(duration / 60);
            const seconds = duration % 60;
            console.log(`⏱️ Duration: ${minutes}:${seconds.toString().padStart(2, '0')}`);
        }

        // Create download content
        const urlContent = videoUrl;
        const summaryContent = `# Skool Loom Video
# Course ID: ${courseId}
# Title: ${title}
# Duration: ${duration ? `${Math.floor(duration/60)}:${(duration%60).toString().padStart(2, '0')}` : 'Unknown'}
# Extracted: ${new Date().toLocaleString()}

${videoUrl}

# To download:
# yt-dlp "${videoUrl}"`;

        // Download function
        function download(content, filename) {
            const blob = new Blob([content], { type: 'text/plain' });
            const url = URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url;
            a.download = filename;
            document.body.appendChild(a);
            a.click();
            document.body.removeChild(a);
            URL.revokeObjectURL(url);
        }

        // Create filename based on title (sanitized)
        const safeTitle = title.replace(/[^a-z0-9]/gi, '_').toLowerCase();
        const shortTitle = safeTitle.substring(0, 30);

        // Download files
        download(urlContent, `loom_${shortTitle}.txt`);
        setTimeout(() => {
            download(summaryContent, `loom_${shortTitle}_info.txt`);
        }, 500);

        // Success message
        setTimeout(() => {
            alert(`✅ Found video: "${title}"\n\nDownloaded:\n• loom_${shortTitle}.txt\n• loom_${shortTitle}_info.txt\n\nTo download:\nyt-dlp --batch-file loom_${shortTitle}.txt`);
        }, 1000);

    } catch (e) {
        console.error('❌ Error parsing page data:', e);
        alert('Error parsing page data. See console for details.');
    }
})();
```

This will download something to your desktop on Mac.

### Step 3: Get the command from the file and run it in your terminal

It will look something like this:

```
# Skool Loom Video
# Course ID: c22bf3e55b6f41f98e12eedfbb14391e
# Title: How We Notify You If You Won
# Duration: 1:23
# Extracted: 5/28/2025, 2:13:17 PM

https://www.loom.com/share/b19b38d0ce2b44f5a049b871d9a38925

# To download:
# yt-dlp "https://www.loom.com/share/b19b38d0ce2b44f5a049b871d9a38925"
```

Focus on the `yt-dlp "https://www.loom.com/share/b19b38d0ce2b44f5a049b871d9a38925"` part.

Paste that in your terminal and it will download to where you are!

So if that helped, and you're feelin generous - feel free to sponsor me so I can make more tutorials like this.

....and take baths like this

![batg](https://gist.github.com/user-attachments/assets/27fc5d1f-e0cd-4c4d-ac24-f8adb27638cc)


> [Click here to sponsor devin's baths](https://serp.ly/@devin/sponsor?via=github&utm_source=github&utm_medium=piggyback&utm_campaign=howtodownloadvideosagent)

# How to download skool.com videos (and the entire course)
## Version 1 - Comprehensive, More Difficult

> You can download the entire classroom with this, but its not beginner friendly.
> If you want the beginner version to just download 1 video at a time, scroll down to the bottom!

This is not a beginner friendly video, but if you follow along I’ll show you how to get around the download protections.

If you’re looking for a simpler solution or have any questions about this process, just leave a comment and I’ll create a follow-up video.

## Watch the video

[![How to Download Videos from Skool.com tutorial thumbnail](https://raw.githubusercontent.com/devinschumacher/uploads/refs/heads/main/images/how-to-download-videos-from-skoolcom-2025-update-step-by-step-tutorial.jpg)](https://www.youtube.com/watch?v=_NinMA9Tqis)

## Prereqs

1. [homebrew](https://brew.sh/) (free)
2. wget (free)
3. [Get cookies.txt LOCALLY](https://chromewebstore.google.com/detail/get-cookiestxt-locally/cclelndahbckbenkjhflpdbgdldlbecc)
4. [visual studio code](https://code.visualstudio.com/) (free)
5. [live server extension](https://marketplace.visualstudio.com/items?itemName=ritwickdey.LiveServer) for visual studio code (free)
6. [pants](https://serp.ly/amazon/pants?via=github&utm_source=github&utm_medium=piggyback&utm_campaign=howtodownloadvideosagent) (in order for this process to work you have to be wearing pants unfortunately)

## Steps overview

1. Put on pants 👖
2. Login to skool & navigate to the classroom
3. Save the cookies with Get cookies.txt extension (netscape format)
4. Open browser console & paste in the `.js` code, press enter
5. Create a folder on your desktop called `project`
6. Put your `cookies.txt` and `urls.txt` file in the project folder
7. Open terminal application and type `cd ~/Desktop/project`
8. Run the `wget` command to start the process
9. Remove pants 🩲 and wait for it to finish


## Detailed step by step guide

### 1. Put on pants 👖

![pants](https://gist.github.com/user-attachments/assets/9f8ad92a-2304-4042-b694-0da153c7db5c)


### 2. Login to skool & navigate to the classroom


### 3. Save the cookies with Get cookies.txt extension (netscape format)


### 4. Open browser console & paste in the `.js` code, press enter


```js
// Extract all lesson URLs from Skool classroom - paste in browser console
function extractAllLessonUrls() {
    console.log("🔍 Searching for lesson URLs on this page...");

    const urls = new Set();

    // Dynamically get base URL from current page
    const currentUrl = new URL(window.location.href);
    const baseUrl = `${currentUrl.protocol}//${currentUrl.host}${currentUrl.pathname}`;

    console.log(`📍 Auto-detected base URL: ${baseUrl}`);
    console.log(`🌐 Current full URL: ${window.location.href}`);

    // Method 1: Look for direct links with md= parameters
    document.querySelectorAll('a[href*="classroom"]').forEach(link => {
        if (link.href.includes('md=')) {
            urls.add(link.href);
            console.log("Found link:", link.href);
        }
    });

    // Method 2: Look for any links containing md= in any attribute
    document.querySelectorAll('*').forEach(el => {
        // Check onclick handlers
        const onclick = el.getAttribute('onclick');
        if (onclick && onclick.includes('md=')) {
            const mdMatch = onclick.match(/md=([a-f0-9]+)/);
            if (mdMatch) {
                const url = `${baseUrl}?md=${mdMatch[1]}`;
                urls.add(url);
                console.log("Found in onclick:", url);
            }
        }

        // Check data attributes
        ['data-href', 'data-url', 'data-link', 'data-md'].forEach(attr => {
            const value = el.getAttribute(attr);
            if (value && value.includes('md=')) {
                if (value.startsWith('http')) {
                    urls.add(value);
                } else {
                    const mdMatch = value.match(/md=([a-f0-9]+)/);
                    if (mdMatch) {
                        const url = `${baseUrl}?md=${mdMatch[1]}`;
                        urls.add(url);
                        console.log(`Found in ${attr}:`, url);
                    }
                }
            }
        });

        // Check for md= in any attribute value
        for (let attr of el.attributes) {
            if (attr.value.includes('md=') && !attr.name.startsWith('data-react')) {
                const mdMatch = attr.value.match(/md=([a-f0-9]+)/g);
                if (mdMatch) {
                    mdMatch.forEach(match => {
                        const mdValue = match.replace('md=', '');
                        const url = `${baseUrl}?md=${mdValue}`;
                        urls.add(url);
                        console.log(`Found in ${attr.name}:`, url);
                    });
                }
            }
        }
    });

    // Method 3: Look in page source/innerHTML for md= patterns
    const pageSource = document.documentElement.innerHTML;
    const mdMatches = pageSource.match(/md=([a-f0-9]{32})/g);
    if (mdMatches) {
        mdMatches.forEach(match => {
            const mdValue = match.replace('md=', '');
            const url = `${baseUrl}?md=${mdValue}`;
            urls.add(url);
        });
        console.log(`Found ${mdMatches.length} md= patterns in page source`);
    }

    // Add current page URL if it has md=
    if (window.location.href.includes('md=')) {
        urls.add(window.location.href);
        console.log("Added current page:", window.location.href);
    }

    const urlArray = Array.from(urls).sort();

    console.log(`\n✅ Found ${urlArray.length} unique lesson URLs:`);
    urlArray.forEach((url, index) => {
        console.log(`${index + 1}. ${url}`);
    });

    if (urlArray.length === 0) {
        console.log("\n❌ No lesson URLs found!");
        console.log("💡 Try running this from the main classroom navigation page");
        console.log("💡 Or manually add URLs to the list below");

        // Add current URL as fallback
        urlArray.push(window.location.href);
    }

    // Create urls.txt file and download it
    const urlText = urlArray.join('\n');
    const blob = new Blob([urlText], {type: 'text/plain'});
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = 'urls.txt';
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);

    console.log("\n📁 Downloaded urls.txt with all found URLs");
    console.log("📝 Next steps:");
    console.log("1. Check the downloaded urls.txt file");
    console.log("2. Add any missing URLs manually if needed");
    console.log("3. Use: wget --input-file=urls.txt [other options]");

    return urlArray;
}

// Run the extraction
extractAllLessonUrls();
```

### 5. Create a folder on your desktop called `project`

### 6. Put your `cookies.txt` and `urls.txt` file in the project folder

### 7. Open terminal application and navigate to your project folder


```bash
cd ~/Desktop/project
```

### 8. Run the `wget` command to start the process


```bash
#!/bin/bash

# Complete website scraping with wget
wget \
  --input-file=urls.txt \
  --page-requisites \
  --html-extension \
  --convert-links \
  --restrict-file-names=windows \
  --force-directories \
  --no-clobber \
  --directory-prefix=./skool_complete \
  --load-cookies=cookies.txt \
  --user-agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36" \
  --wait=2 \
  --span-hosts \
  --recursive \
  --level=5 \
  --no-parent
```


### 9. Remove pants 🩲 and wait for it to finish

![2](https://gist.github.com/user-attachments/assets/d9889141-4333-42d2-ac87-b9498d4ce182)


### 10. How to use it

At this point you basically have a copy of what school has on their servers but you have it on your computer so you can access it anytime anywhere and without internet. Simply open the folder in Visual Studio Code and click on one of the HTML files and open with live server.


---

# How to download a single video from skool.com (easy method)

> Here's the easier version, as request by all ye youtubers!


## Step 1: Visit the page on skool.com where the video is

## Step 2: Copy and paste this command in your dev tools console (right click > inspect > go to console tab)

```js
/**
 * Skool Loom Video Extractor - Final Version
 * Extracts video from current lesson using the md query parameter
 *
 * Usage: Paste this entire script in browser console while on a Skool classroom page
 * Then, go subscribe to @devinschumacher on YouTube
 */

(function() {
    console.log('🎥 Skool Loom Video Extractor - Final Version');

    // Get the md parameter from URL
    const urlParams = new URLSearchParams(window.location.search);
    const courseId = urlParams.get('md');

    if (!courseId) {
        console.log('❌ No "md" parameter found in URL');
        alert('This page does not have a lesson ID (md parameter). Make sure you are on a lesson page.');
        return;
    }

    console.log(`📚 Looking for course with ID: ${courseId}`);

    // Function to recursively search for course by ID
    function findCourseById(obj, targetId) {
        if (!obj || typeof obj !== 'object') return null;

        // Direct match
        if (obj.id === targetId && obj.metadata && obj.metadata.videoLink) {
            return obj;
        }

        // Check if this object has a course property with matching ID
        if (obj.course && obj.course.id === targetId && obj.course.metadata && obj.course.metadata.videoLink) {
            return obj.course;
        }

        // Recursively search all properties
        for (let key in obj) {
            if (obj.hasOwnProperty(key)) {
                const result = findCourseById(obj[key], targetId);
                if (result) return result;
            }
        }

        return null;
    }

    // Get the Next.js data
    const nextDataScript = document.getElementById('__NEXT_DATA__');
    if (!nextDataScript) {
        console.log('❌ Could not find __NEXT_DATA__ script');
        alert('Could not find page data. This script works on Skool classroom pages.');
        return;
    }

    try {
        const nextData = JSON.parse(nextDataScript.textContent);
        console.log('📋 Found page data, searching for course...');

        // Search for the course
        const course = findCourseById(nextData, courseId);

        if (!course) {
            console.log('❌ Could not find course with ID:', courseId);
            alert(`Could not find lesson with ID: ${courseId}`);
            return;
        }

        // Extract video information
        const metadata = course.metadata;
        const videoUrl = metadata.videoLink.split('?')[0]; // Clean URL without parameters
        const title = metadata.title || 'Untitled Lesson';
        const duration = metadata.videoLenMs ? Math.round(metadata.videoLenMs / 1000) : null;

        console.log('✅ Found video!');
        console.log(`📹 Title: ${title}`);
        console.log(`🔗 URL: ${videoUrl}`);
        if (duration) {
            const minutes = Math.floor(duration / 60);
            const seconds = duration % 60;
            console.log(`⏱️ Duration: ${minutes}:${seconds.toString().padStart(2, '0')}`);
        }

        // Create download content
        const urlContent = videoUrl;
        const summaryContent = `# Skool Loom Video
# Course ID: ${courseId}
# Title: ${title}
# Duration: ${duration ? `${Math.floor(duration/60)}:${(duration%60).toString().padStart(2, '0')}` : 'Unknown'}
# Extracted: ${new Date().toLocaleString()}

${videoUrl}

# To download:
# yt-dlp "${videoUrl}"`;

        // Download function
        function download(content, filename) {
            const blob = new Blob([content], { type: 'text/plain' });
            const url = URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url;
            a.download = filename;
            document.body.appendChild(a);
            a.click();
            document.body.removeChild(a);
            URL.revokeObjectURL(url);
        }

        // Create filename based on title (sanitized)
        const safeTitle = title.replace(/[^a-z0-9]/gi, '_').toLowerCase();
        const shortTitle = safeTitle.substring(0, 30);

        // Download files
        download(urlContent, `loom_${shortTitle}.txt`);
        setTimeout(() => {
            download(summaryContent, `loom_${shortTitle}_info.txt`);
        }, 500);

        // Success message
        setTimeout(() => {
            alert(`✅ Found video: "${title}"\n\nDownloaded:\n• loom_${shortTitle}.txt\n• loom_${shortTitle}_info.txt\n\nTo download:\nyt-dlp --batch-file loom_${shortTitle}.txt`);
        }, 1000);

    } catch (e) {
        console.error('❌ Error parsing page data:', e);
        alert('Error parsing page data. See console for details.');
    }
})();
```

This will download something to your desktop on Mac.

### Step 3: Get the command from the file and run it in your terminal

It will look something like this:

```
# Skool Loom Video
# Course ID: c22bf3e55b6f41f98e12eedfbb14391e
# Title: How We Notify You If You Won
# Duration: 1:23
# Extracted: 5/28/2025, 2:13:17 PM

https://www.loom.com/share/b19b38d0ce2b44f5a049b871d9a38925

# To download:
# yt-dlp "https://www.loom.com/share/b19b38d0ce2b44f5a049b871d9a38925"
```

Focus on the `yt-dlp "https://www.loom.com/share/b19b38d0ce2b44f5a049b871d9a38925"` part.

Paste that in your terminal and it will download to where you are!

So if that helped, and you're feelin generous - feel free to sponsor me so I can make more tutorials like this.

....and take baths like this

![batg](https://gist.github.com/user-attachments/assets/27fc5d1f-e0cd-4c4d-ac24-f8adb27638cc)


> [Click here to sponsor devin's baths](https://serp.ly/@devin/sponsor?via=github&utm_source=github&utm_medium=piggyback&utm_campaign=howtodownloadvideosagent)
