

# Push CDNs
The origin server actively uploads new content to CDN. 

Pros:
- There will be decreased traffic between origin and cdn, as content is uploaded only when the content has a newer version.
- Suitable for large files (like OS images, Software distribution etc).
- Suiltable for cases when a piece of data doesn't change frequently. You upload a data, it stays on CDN for a longer perid of time compared to pull model.

Cons:
- Storage on CDN will be high, as we are actively uploading all content to the CDN, even when the user has not requested for it.
- Not suitable for cases when the data changes frequently. It will put a lot of stress on the origin server.
- May have to upload data manually.


# Pull CDNs
The CDN edge servers fetches content from the origin (if not cached) **on-demand** by the user.

Pros:
- Less storage required on CDN servers, as only those data is stored there that were actually requested.
- Automatically fetch content from the origin, if the requested content in the cache has expired. No need to manually upload data from origin to CDN.

Cons:
- Increased traffic between origin and CDN. Imagine the TTL for a content expires and it gets re-fetched, even if it has not changed. So, for the same piece of data, its being fetched again after a while, which eventually increases traffic.
- A user might have to wait a while to get the latest data, as the data will be re-fetched only when the TTL expires.
