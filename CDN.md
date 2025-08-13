

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
- Increased traffic between origin and CDN. Imagine the TTL for a content expires (or TTL is very short) and it gets re-fetched, even if it has not changed. So, for the same piece of data, its being fetched again after a while, which eventually increases traffic.
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

