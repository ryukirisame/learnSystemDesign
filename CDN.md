
https://blog.algomaster.io/p/content-delivery-networks

# Push CDNs
The origin server actively uploads new content to CDN in advance.

Pros:
- There will be decreased traffic between origin and cdn, as content is uploaded only when the content has a newer version.
- Suitable for large files (like OS images, Software distribution etc).
- Suiltable for cases when a piece of data doesn't change frequently. You upload a data, it stays on CDN for a longer perid of time compared to pull model.
- Longer cache duration: Since content is proactively pushed, it stays cached until explicitly updated or purged.

Cons:
- Storage on CDN will be high, as we are actively uploading all content to the CDN, even when the user has not requested for it.
- Not suitable for cases when the data changes frequently. It will put a lot of stress on the origin server.
- May have to upload data manually.


# Pull CDNs
The CDN edge servers fetches content from the origin (if not cached) **on-demand**, when requested by the user.

Pros:
- Less storage required on CDN servers, as only those data is stored there that were actually requested.
- Automatically fetches content from the origin, if the requested content in the cache has expired. No need to manually upload data from origin to CDN.
- Dynamic content support: Easier to work with sites where content changes regularly

Cons:
- Increased traffic between origin and CDN. Imagine the TTL for a content expires (or TTL is very short) and it gets re-fetched, even if it has not changed. So, for the same piece of data, its being fetched again after a while, which eventually increases traffic. (NOTE: This can be optimized, we'll see later)
- A user might have to wait a while to get the latest data, as the data will be re-fetched only when the TTL expires.
- If content is not cached and origin is down, the request will fail.


# Use Cases
## 🌐 **Pull CDN — Real-World Use Cases**

Pull CDNs shine where content changes often or you don’t want to manually manage uploads.

| Industry / Scenario                       | Example                                    | Why Pull Works Well                                                                                                    |
| ----------------------------------------- | ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| **News & Media Websites**                 | BBC News, The Guardian                     | Articles, images, and videos are constantly updated — CDN automatically fetches latest content without manual uploads. |
| **E-commerce**                            | Amazon, Flipkart                           | Product images, descriptions, and availability change frequently — Pull ensures only requested assets are cached.      |
| **Blogs / CMS-based Sites**               | Medium, WordPress.com                      | Content is added or updated often — Pull fetches only what users access.                                               |
| **Streaming Thumbnails / Dynamic Images** | Netflix preview images, YouTube thumbnails | These change often based on algorithms and recommendations — Pull adapts without manual push.                          |
| **SaaS Web Apps**                         | GitHub, Trello                             | Static files (JS/CSS) are cached, but dynamic API-driven pages are served fresh — Pull fits naturally.                 |



## 📦 **Push CDN — Real-World Use Cases**

Push CDNs shine where files are large, infrequently updated, and you want maximum control over delivery.

| Industry / Scenario       | Example                                               | Why Push Works Well                                                                                    |
| ------------------------- | ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| **Software Downloads**    | Microsoft Windows ISO, Ubuntu images                  | Huge files rarely change — pushing them ensures they’re pre-cached globally and instantly available.   |
| **Game Updates & Assets** | Steam, Epic Games Store                               | Game patches and assets are large and versioned — pushed once, then served millions of times.          |
| **Video on Demand (VOD)** | Netflix offline download files, Apple TV movie assets | Large pre-encoded video segments stored in CDN for instant access.                                     |
| **Mobile App Stores**     | Google Play Store, Apple App Store                    | APK/IPA files are large, updated periodically — pushing ensures global readiness before release.       |
| **AR/VR Content**         | Oculus/Meta VR experiences                            | 3D models and textures are huge — pushing ensures edge nodes have them before users start downloading. |


## 🧠 Quick Way to Remember:

* **Pull CDN** = “Lazy Loading” — fetch when needed. Great for *changing or unpredictable* content.
* **Push CDN** = “Preloading” — upload everything in advance. Great for *big, rarely changing* files.

# Additional Notes

## TTL (Time to Live) in Pull CDNs

TTL tells the CDN how long it should keep a cached file before considering it “expired.”

### How it works in Pull CDNs:
- When TTL expires, the CDN will discard the cached file (or mark it stale).
- On the next user request, the CDN must fetch it again from the origin server.

### Why it's important:
- If TTL is too short → unnecessary re-fetches → more origin traffic.
- If TTL is too long → users might see outdated content.


## Versioning in Push CDNs
- The challenge: In Push CDNs, once you upload a file, it’s stored across all edge servers. If you update it, you have to either:
  - Wait for caches to expire (slow), or
  - Force a cache purge (can be expensive and slow to propagate).

- The solution: Versioning your file names.
  - Example: Instead of logo.png always, use:
    - v1.0/logo.png
    - v1.1/logo.png
  - Each version is treated as a completely new file by the CDN.

- Why it’s useful:
  - Old versions stay available if needed.
  - No risk of users seeing an old cached version.
  - No need for global cache invalidation.




# How to optimize re-fetch in pull CDNs
Pull CDNs don’t blindly re-download files when the cache expires; instead, they often validate with the origin server to check if the file has changed.

##  **What Happens When TTL Expires in a Pull CDN**

1. **User requests** a file.
2. The **edge cache** sees the TTL has expired — file is *stale*.
3. Instead of instantly downloading the whole file again, the CDN sends a **conditional request** to the origin server.
4. The origin server then checks:
   * If the file has **not changed** → sends a **`304 Not Modified`** response (no content body).
   * If the file **has changed** → sends a **`200 OK`** with the new file content.
5. The CDN updates or refreshes its cache accordingly.


##  **How the CDN Knows if the File Changed**

It's through some **HTTP headers**:

### 1. **ETag** - Response header
* Sent by origin to CDN included in the response the first time the content is fetched.
* Think of it as a file fingerprint (hash or unique ID).
* CDN stores this value with the cached file.
* Changes when the content/file changes.
* 
* https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/ETag

### 2. If-None-Match - Request header
- Corresponding request header of ETag sent by CDN to origin.
- Makes a request conditional.
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/If-None-Match

### 3. **Last-Modified** - Response header
* Timestamp of when the file was last changed.
* CDN stores this date/time.
* https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Last-Modified

### 4. If-Modified-Since - Request header
- Corresponding request header of Last-Modified.
- Sent by CDN to origin.
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/If-Modified-Since


# Full example flow:

We have:

* **Origin server**: `origin.example.com`
* **Pull CDN**: `cdn.example.com`
* **Asset**: `/images/logo.png`
* **TTL in CDN**: 24 hours

## **Step 1 — First Request (Cache Miss)**

A user requests `https://cdn.example.com/images/logo.png`.

**CDN behavior**:

* Checks its edge cache → **miss** (file not stored or expired).
* Sends request to **origin server**.

### **Request from CDN to Origin**

```http
GET /images/logo.png HTTP/1.1
Host: origin.example.com
User-Agent: CDN-Edge-Server
```

### **Response from Origin**

```http
HTTP/1.1 200 OK
Content-Type: image/png
Content-Length: 53217
Cache-Control: max-age=86400
ETag: "abc123xyz"
Last-Modified: Wed, 21 Oct 2015 07:28:00 GMT
```

**Explanation of headers:**

* **`Cache-Control: max-age=86400`** → tells CDN to cache this file for **24 hours**.
* **`ETag`** → unique file fingerprint (string/hash). Changes if file changes.
* **`Last-Modified`** → timestamp when file last changed.
* **`Content-Length`** → file size in bytes.

**CDN action**: Stores:

* File content
* `ETag`
* `Last-Modified`
* Expiry time = now + 24h



## **Step 2 — Subsequent Requests Within TTL**

If a user requests again **before TTL expires**:

* CDN serves directly from edge cache.
* **No request** is sent to origin.
* Response is instant.



## **Step 3 — TTL Expires (Cache Stale)**

After 24h, a new request comes in:

* File is **stale** (TTL expired).
* CDN needs to **revalidate** before serving.


## **Step 4 — Conditional GET to Origin**

Instead of re-downloading the file, CDN sends a **conditional request** using stored `ETag` and/or `Last-Modified`.

### **Conditional Request**

```http
GET /images/logo.png HTTP/1.1
Host: origin.example.com
User-Agent: CDN-Edge-Server
If-None-Match: "abc123xyz"
If-Modified-Since: Wed, 21 Oct 2015 07:28:00 GMT
```

**Explanation:**

* **`If-None-Match`** → asks origin: “Only send file if its ETag is different from `"abc123xyz"`.”
* **`If-Modified-Since`** → asks origin: “Only send file if it was modified after `Wed, 21 Oct 2015 07:28:00 GMT`.”


## **Step 5 — Origin Determines File Status**

Two possibilities:

### **Case A — File Unchanged**

Origin checks:

* Current ETag = `"abc123xyz"` (same as before)
* Last Modified = same date

Returns:

```http
HTTP/1.1 304 Not Modified
Date: Thu, 22 Oct 2015 07:28:00 GMT
Cache-Control: max-age=86400
ETag: "abc123xyz"
Last-Modified: Wed, 21 Oct 2015 07:28:00 GMT
```

**Notes:**

* **`304 Not Modified`** → No body sent, just headers.
* **`ETag`** and **`Last-Modified`** confirm the file is unchanged.
* File is **NOT re-downloaded**, CDN simply extends the cache TTL.

### **Case B — File Changed**

If ETag or Last-Modified has changed, origin sends:

```http
HTTP/1.1 200 OK
Content-Type: image/png
Content-Length: 53280
Cache-Control: max-age=86400
ETag: "def456uvw"
Last-Modified: Fri, 23 Oct 2015 10:12:00 GMT
```

**Notes:**

* `200 OK` → full file sent again.
* `ETag` and `Last-Modified` are updated in CDN cache.


## **Benefits of This Approach**

* **Bandwidth saving**: `304 Not Modified` is tiny (\~few hundred bytes) compared to file download.
* **Speed**: Faster responses for unchanged files.
* **Origin load reduction**: Less CPU, disk, and network usage.


# ETag vs Last-Modified

We don't need to use both of them to check whether the content has been updated on origin. We could use just one of them. But there are trade-offs.

### **1. Last-Modified / If-Modified-Since**

* **What it is**: A timestamp indicating when the resource last changed.
* **Workflow**:

  1. **Origin Server Response** (initial fetch)

     ```
     Last-Modified: Wed, 07 Aug 2024 10:23:45 GMT
     ```
  2. **Next request** (by CDN/browser when cache is stale):

     ```
     If-Modified-Since: Wed, 07 Aug 2024 10:23:45 GMT
     ```
  3. If content is unchanged, server responds with:

     ```
     HTTP/1.1 304 Not Modified
     ```
* **Pros**:
  * Simple and human-readable.
  * Very lightweight.
* **Cons**:
  * Only accurate to **1-second resolution**. Notice the time-stamp, it's only till seconds, not milli-seconds. So, if a change happens quickly within milli-seconds, changes may be missed.
  * If files change more than once in a second, changes may be missed.
  * Can give false positives if server clocks differ.

### **2. ETag / If-None-Match**

* **What it is**: A unique identifier (hash, checksum, or version ID) for the resource content.
* **Workflow**:

  1. **Server Response** (initial fetch):

     ```
     ETag: "abc123xyz"
     ```
  2. **Next request**:

     ```
     If-None-Match: "abc123xyz"
     ```
  3. If ETag matches, server responds:

     ```
     HTTP/1.1 304 Not Modified
     ```
* **Pros**:

  * More precise — detects changes even within the same second.
  * Works even if clocks are out of sync.
  * Can be based on content hash (guarantees accuracy).
* **Cons**:

  * Slightly more overhead (server must compute hash or generate unique version IDs).
  * Poorly implemented ETags (e.g., including machine-specific data in them) can break caching between servers.

### **So why use both sometimes?**

* **Fallback & safety**: Some servers send both to maximize compatibility. (In cases where ETag is not supported, Last-Modified would work and vice versa)
* **Different strengths**:

  * `Last-Modified` → Fast and simple timestamp check.
  * `ETag` → Stronger guarantee of content match.

For a CDN:

* **High-performance sites** often prefer **ETag** for accuracy.
* **Static file servers** sometimes rely only on `Last-Modified` since the timestamp doesn’t change unless you redeploy.

# Conditional Request
A **conditional request** in HTTP is when the client tells the server:

> "Send me the resource **only if** it meets a certain condition."

This condition is usually about whether the resource has changed since the last time the client saw it — so the server can avoid sending the full content if nothing changed.

## 🔹 How it works

A conditional request uses special HTTP request headers that set the condition.
If the condition is **true**, the server sends the full resource (`200 OK`).
If the condition is **false**, the server sends **only a small response** (usually `304 Not Modified`) with no body.




# Extra Resources
- Cloudflare docs: https://developers.cloudflare.com/cache/concepts/revalidation/


