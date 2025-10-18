# HorusApp – AI Crawler + Face-Match

A desktop application that crawls websites for images and videos, downloads media efficiently, and performs streaming face recognition against user-provided reference images. The UI is built with PyQt6 and the crawler supports both direct HTTP downloads and resilient video downloads via yt-dlp. Face matching runs in the background while files are being saved.

---

## Quick Start

1. **Install system dependencies**
   - Ensure Python 3.8+ and ffmpeg are installed and on PATH.
     - Ubuntu/Debian: `sudo apt-get update && sudo apt-get install -y ffmpeg`

2. **Install Python dependencies**
   - From the project root (`HoursApp_18Oct/`):
     ```bash
     pip install -r requirements.txt
     ```

3. **Run the app**
   - From the project root:
     ```bash
     python main.py
     ```

4. **Use the app**
   - In the Search tab, add up to 5 reference face images.
   - Enter up to 5 URLs and per-URL page limits.
   - Click Start to begin crawling and streaming face matching.
   - Media saves to `~/Downloads/Results/CrawledMedia/`.
   - Matched results appear in real time in the UI. Double-click a result to open source.

5. **Where files go**
   - Downloads: `~/Downloads/Results/CrawledMedia/`
   - Matches: `~/Downloads/Results/Face_Matches/`
   - Log: `~/Downloads/Results/HorusApp.log`
   - URL list: `~/Downloads/Results/result_url.txt`
   - Path→link index: `~/Downloads/Results/media_index.jsonl`

---

## Project Structure

```
HoursApp_18Oct/
├─ GUI/                         # UI assets (images, branding)
├─ face_recognition_models/     # dlib models for face recognition
├─ main.py                      # PyQt6 app entrypoint & UI logic
├─ frame_pages.py               # UI pages/components
├─ face_match.py                # Face loading & matching utilities
├─ antibot_crawl_utils.py       # Driver stealth/proxy helpers
├─ multi_crawler.py             # Full-featured crawler (HTTP + Selenium + yt-dlp)
├─ multi_crawler_Ydl.py         # Lean crawler with generic yt-dlp fallback
├─ video_crawler.py             # Additional crawler (legacy/aux)
├─ requirements.txt             # Python deps
└─ docs/
   └─ README.md                 # This document
```

---

## Key Components

- `main.py` (PyQt6)
  - Search, Settings, History, Help tabs.
  - Coordinates background threads: `MultiCrawlWorker` (crawl) and `FaceMatchWorker` (streaming face-match).
  - Real-time progress bars per URL; cooperative Stop.

- `multi_crawler.py` (Full crawler)
  - Static and dynamic extraction (BeautifulSoup, Selenium fallback) + robust yt-dlp integration.
  - Expanded `SPECIAL_VIDEO_HOSTS` for platforms like `rumble.com`, `odysee.com`, `bitchute.com`, `twitch.tv`, etc.
  - Skips non-media/static assets (fonts/CSS/JS/JSON) and avoids calling yt-dlp on them.
  - Caching, rate limiting, connection pooling, concurrent per-URL crawling.
  - Writes `result_url.txt` and `media_index.jsonl` to map saved file paths to original links.

- `multi_crawler_Ydl.py` (Lean crawler)
  - Direct HTTP downloads + generic yt-dlp fallback on HTML pages.
  - Non-media/static assets filtered out.

- `face_match.py`
  - `load_reference_faces(paths)` loads multiple reference faces.
  - `match_faces_in_paths()` processes images and videos; used live by the streaming worker.
  - `get_original_link_for_path(local_path)` resolves back to the original URL via `media_index.jsonl`.

- `antibot_crawl_utils.py`
  - Undetected Chrome setup, stealth injection, user agent randomization, window sizing, proxy support.

---

## Crawler Flow (multi_crawler.py)

1. `multi_crawler(url_limit_pairs, progress_callback, per_file_callback, stop_checker, use_proxy)`
   - Orchestrates concurrent crawling of up to 5 URLs using `ThreadPoolExecutor`.

2. `crawl_single_url((idx, url, max_pages, ...))`
   - BFS of same-domain links.
   - Extracts media:
     - Static DOM: `extract_media_links_static()` scans `img`, `video`, `source`, `picture source[srcset]`, inline `style`, computed `background-image`, `meta og:image`, `link image_src/preload/icon`, and `iframe[src]`.
     - Text: `extract_media_links_from_text()` scans scripts/JSON for direct URLs.
     - JS fallback: `extract_media_links_js()` via Selenium driver pool when few links found.
   - Concurrent downloads with cooperative STOP and per-file callback.

3. `OptimizedCrawler.download_and_save_sync(link, referer, proxies, on_saved, stop_checker)`
   - Stop-aware cache check and static asset filter.
   - Special video hosts → resilient yt-dlp first (unless asset is a font/CSS/JS/JSON).
   - HTTP streaming for direct images/videos; `.part` temp → atomic rename.
   - If not direct media and content-type is HTML-like (or unknown), generic yt-dlp fallback.
   - On save: append to `result_url.txt` and record in `media_index.jsonl`, emit `on_saved`.

---

## File Naming and Save Path

- All media are saved directly under `~/Downloads/Results/CrawledMedia/` (no subdirectories).
- Filename pattern: `{domain}_{basename}{ext}`.
- If basename is missing/ambiguous, a hash-backed filename is generated.
- Content-Disposition filenames are respected (prefixed by domain for uniqueness).

---

## Proxy Support

- Configure proxies in `Settings` tab (GUI). Values are saved to `config.json`.
- Supported formats (one per line):
  - `host:port`
  - `host:portStart-portEnd` (port range)
  - `A.B.C.X-Y:port` (IP last-octet range)
  - Optional auth: `http://user:pass@host:port`
- `ProxyManager` normalizes schemes and provides proxies for requests, Selenium, and yt-dlp.

---

## Streaming Face Match

- `MultiCrawlWorker.file_saved` emits each saved file path.
- `FaceMatchWorker.process_path()` consumes paths and runs `match_faces_in_paths()`.
- Results are emitted to the UI per match without waiting for the full crawl to finish.

---

## Cancellation

- STOP button sets a shared flag observed by:
  - Download loops (per-chunk checks + `.part` cleanup on cancel)
  - yt-dlp progress hooks (raise to abort)
  - Selenium fallback (pre-check before expensive steps)
  - ThreadPoolExecutor (cancel pending futures)

---

## yt-dlp & ffmpeg Settings

- We do not force `mp4`; container chosen by yt-dlp/ffmpeg to reduce merge errors.
- Preferred format: `bv*+ba/b` (best video+audio, fallback to best single stream).
- `keepvideo=True` keeps separate files if merges fail, to avoid data loss.
- `restrictfilenames=True` for safe, portable filenames.
- Temp paths are pinned to `SAVE_DIR` so `/tmp` pressure does not break merges.

> Tip: Ensure enough free disk space (10–20GB recommended for large videos). Low space can cause `ffmpeg` exit code 255.

---

## Troubleshooting

- **No matches found**
  - Use clear, frontal reference faces. Add multiple references.

- **Crawling is slow**
  - Reduce per-URL page limits. Disable Selenium fallback by ensuring static extraction returns enough links.

- **ffmpeg exited with code 255**
  - Free disk space. We keep separate streams on merge failure, but merges still require space.

- **Unsupported URL (fonts/CSS/JS/JSON)**
  - These are filtered and skipped; ensure your input URL is a page or direct media.

- **Blocked by site**
  - Enable proxies in Settings. Consider increasing rate limit delay.

---

## Screenshots (placeholders)

- Add GUI screenshots under `docs/images/` and update the links below:

![Welcome](images/welcome.png)
![Search](images/search.png)
![Settings](images/settings.png)
![Results](images/results.png)

---

## Contributing & Versioning

- Update this `docs/README.md` when changing core crawler logic or UI workflows.
- Consider adding CHANGELOG entries for major behavioral changes (extraction rules, save paths, etc.).

---

## License

- Internal/Proprietary (update this section as needed).
# AI-Video-Image-Recognition-Application-Development
