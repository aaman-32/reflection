# Reflection

**Reflection** is an open-source Express.js API for searching media metadata, processing YouTube audio/video downloads, and uploading images. It provides a small HTTP interface over third-party services.

## Contents

- [Features](#features)
- [Requirements](#requirements)
- [Installation](#installation)
- [Configuration](#configuration)
- [Starting the API](#starting-the-api)
- [Endpoints](#endpoints)
- [Errors](#errors)
- [Notes](#notes)

# Features

- Search music using Deezer.
- Search YouTube videos.
- Search movies using The Movie Database (TMDB).
- Download YouTube, Instagram, Facebook, TikTok and Pinterest media.
- Upload images through ImgBB.
- JSON responses and standard HTTP status codes for errors.

## Requirements

- Node.js **v24 or later**.
- npm.
- `ffmpeg` installed and available on `PATH` for audio extraction and media merging.
- A **local server** or **VPS** (Render is recommended)
- **API keys** for the providers used by the relevant endpoints (see [environment values](#1-environment-values)).

## Installation

```bash
git clone https://github.com/amanumar32/reflection.git
cd reflection
npm install
```

## Configuration

### 1. Environment values
Some features like image uploading an movie searching require an API key. The services used are completely free and have almost unlimited free tier limits. You can however use the API for free, but some endpoints might not function the way they are supposed to.

The following API keys can be configured:

| Key | Requirement | Where to find |
| --- | --- | --- |
| `TMDB_API_KEY` | Movie search | [The Movie Database](https://www.themoviedb.org/settings/api) |
| `IMGBB_API_KEY` | Image upload | [ImgBB](https://api.imgbb.com/) |

Create a `.env` file and add the keys, or inject the values to your VPS environment variables service.
```dotenv
TMDB_API_KEY="xxxx"
IMGBB_API_KEY="xxxx"
```

### 2. Configs

You can edit the `configs` object in [app.js](./app.js#16) to match your preferences:

| Key | Description | Default | Notes |
| --- | --- | --- | --- |
| `max_process_limit` | The maximum number of concurrent requests that can run at a time. | 1 | Too many concurrent processes can crash your server. |
| `max_file_size` | The maximum file size the API can upload (in kilobytes). | 104857600 | The third-party upload services may have their own limits and reject your uploads if your limit surpasses theirs. |
| `port` | The port to listen on. | 8080 | If you're using a VPS, the port value provided by the will be treated as a priority. |

```js
const configs = {
    max_process_limit: 1,
    max_file_size: 104857600,
    port: 8080
}
```

### 3. Cookies
The youtube downloader, `youtube-dl-exec`, may run into bot verification issues while downloading your files. You can bypass these checks completely by including cookies. To add cookies:

- Prepare a disposable Google account you won't be using anytime after this.
- Install the `Get cookies.txt LOCALLY` extension on your PC's browser.
- Go to **the extensions page** on your browser and turn on **"Allow in Incognito"** (or any similar option) to allow the extension run in incognito mode.
- Open a new **incognito** tab on your browser.
- Go to [YouTube](https://www.youtube.com) and login to your account.
- In the same tab, go to the [YouTube bots page](https://www.youtube.com/robots.txt).
- Open your `Get cookies.txt LOCALLY` extension and ensure the format is **Netscape**. Click **Export** and export your cookies to `cookies.txt`.
- **IMPORTANT:** Immediately close your incognito tab. **Do Not logout of YouTube**, and ensure you do not login to the account again, unless you're adding new cookies.
- Paste the `cookies.txt` file in your project directory, **_I'll handle the rest_**.

## Starting the API

The way you start the API depends on how you're deploying it.
### 1. VPS
Most VPS services inject variables automatically for you. If you've specified your variables on your VPS dashboard, run the `start` script to avoid conflicts:
```bash
npm run start
```
Your server will give you the endpoint to your live API.

### 2. Locally
If you're running locally, you'll need to inject the values yourself. Run the `dev` script to start the API with your values:
```bash
npm run dev
```
Your api should be available on [localhost:8080](http://localhost:8080/api) (or whatever your port is).

## Endpoints
The following are the available endpoints, their request types and example JSON:

### 1. Health check
`GET /api`

Checks whether the API server is running.

**Example request**

```http
GET /api
```

**Example response — `200 OK`**

```json
{
  "message": "API is up and running!"
}
```

### 2. Search
`GET /api/search/:type?query=...`

Searches a supported media catalogue.

**Path parameter**

| Parameter | Accepted values | Description |
| --- | --- | --- |
| `type` | `music`, `yt`, `movie` | Selects the search provider and result format. |

**Query parameters**

| Parameter | Required | Description |
| --- | --- | --- |
| `query` | Yes | Search terms, such as a song title, artist, YouTube video, or movie title. |

#### Search types

- `music` — searches Deezer.
- `yt` — searches YouTube videos.
- `movie` — searches TMDB. Requires `TMDB_API_KEY`.

#### Example: music search

```http
GET /api/search/music?query=Daft%20Punk
```

**Example response — `200 OK`**

```json
{
  "mode": "search",
  "type": "music",
  "query": "Daft Punk",
  "results": [
    {
      "id": 123456,
      "title": "Example Track",
      "preview": "https://example.com/preview.mp3",
      "thumbnail": "https://example.com/album-cover.jpg",
      "artist": "Example Artist",
      "album": "Example Album"
    }
  ],
  "length": 1
}
```

#### Example: YouTube search

```http
GET /api/search/yt?query=lofi%20music
```

**Example response — `200 OK`**

```json
{
  "mode": "search",
  "type": "yt",
  "query": "lofi music",
  "results": [
    {
      "id": "abcdefghijk",
      "title": "Example YouTube Video",
      "author": "Example Channel",
      "url": "https://www.youtube.com/watch?v=abcdefghijk",
      "thumbnail": "https://example.com/thumbnail.jpg",
      "description": "Example video description"
    }
  ],
  "length": 1
}
```

#### Example: movie search

```http
GET /api/search/movie?query=Inception
```

**Example response — `200 OK`**

```json
{
  "mode": "search",
  "type": "movie",
  "query": "Inception",
  "results": [
    {
      "id": 27205,
      "title": "Inception",
      "overview": "Example movie overview.",
      "release_date": "2010-07-16",
      "rating": 8.4,
      "poster": "https://image.tmdb.org/t/p/w500/example.jpg"
    }
  ],
  "length": 1
}
```

### 3. Download
`GET /api/download/:type?query=...&media=...`

Searches for or accepts a URL, processes the selected media, uploads the resulting file to a temporary file host, and returns a URL to the hosted file.

**Path parameter**

| Parameter | Accepted values | Description |
| --- | --- | --- |
| `type` | `yt` | Selects the download provider. |

**Query parameters**

| Parameter | Required | Default | Description |
| --- | --- | --- | --- |
| `query` | Yes | — | A URL/video link or search phrase. Search phrases resolve to the first search result. |
| `media` | No | `video` | `video` returns MP4 video; `audio` extracts MP3 audio. |

#### Example: download video

```http
GET /api/download/yt?query=https%3A%2F%2Fwww.youtube.com%2Fwatch%3Fv%3Dabcdefghijk&media=video
```

**Example response — `200 OK`**

```json
{
  "mode": "download",
  "type": "yt",
  "query": "https://www.youtube.com/watch?v=abcdefghijk",
  "media": "video",
  "title": "",
  "thumbnail": "",
  "mimetype": "video/mp4",
  "size": 12345678,
  "url": "https://files.example.com/example.mp4",
  "expires": 1790000000000
}
```

#### Example: download audio

```http
GET /api/download/yt?query=artist%20song%20name&media=audio
```

**Example response — `200 OK`**

```json
{
  "mode": "download",
  "type": "yt",
  "query": "artist song name",
  "media": "audio",
  "title": "Example Song",
  "thumbnail": "https://example.com/thumbnail.jpg",
  "mimetype": "audio/mpeg",
  "size": 3456789,
  "url": "https://files.example.com/example.mp3",
  "expires": 1790000000000
}
```

### 4. Upload
`POST /api/upload/:type`

Uploads an image or file to ImgBB or a temporary host respectively.

**Path parameter**

| Parameter | Accepted values | Description |
| --- | --- | --- |
| `type` | `image`, `file` | Selects the upload provider. |

#### Example request

```bash
curl -X POST http://localhost:8080/api/upload/image \
  -F "image=@./picture.jpg"
```

**Example response — `200 OK`**

```json
{
  "mode": "upload",
  "mimetype": "image/jpeg",
  "url": "https://i.ibb.co/example/picture.jpg",
  "size": 456789,
  "expires": null
}
```

## Errors

Errors are returned as JSON with an `error` message.

**Example — missing `query` (`400 Bad Request`)**

```json
{
  "error": "Query parameter is required"
}
```

**Example — unsupported type (`400 Bad Request`)**

```json
{
  "error": "Invalid type parameter"
}
```

Common error cases include:

| HTTP status | Meaning / example |
| --- | --- |
| `400` | Missing required query/file, unsupported search or download type, or unsupported upload type. |
| `404` | No media found for a search. |
| `500` | Unexpected server/provider failure, missing required environment configuration, or a media processing/upload failure. |

The exact error message depends on the endpoint and the upstream service. Some upstream failures may be returned as `500`.

### Notes

- **Temporary files:** Downloaded media is uploaded to an external temporary host; it is not stored permanently by Reflection.
- **Download size:** The upload helper may reject files larger than the range **100 MB - 1 GB**.
- **Concurrency:** Download processing is queued with a concurrency limits.
- **Third-party dependencies:** Search, downloading, and hosting rely on external services that may change, rate-limit requests, or become unavailable.
- **YouTube access:** Downloads may fail when YouTube blocks a request or requires additional verification.
- **Image upload:** Requires a valid ImgBB API key.
- **CORS:** CORS is enabled in the current source.

**Hope you enjoy our project!**

**Maintained by [Halo Studios](https://halostudios.xyz). Copyleft (c) 2026 Halo Studios**

