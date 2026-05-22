# Design Spec: Speechify-Style iOS Audiobook App with Voice Cloning

**Date:** 2026-05-22
**Status:** Approved
**Distribution:** Private — 2 users (owner + 1 friend via AltStore sideload)

---

## Overview

A two-part system: an iOS app (React Native / Expo) that lets users photograph a book title, automatically find and download the free PDF, then listen to it read aloud in a cloned celebrity voice. The Mac mini acts as the backend server for voice cloning, book search, and audio synthesis. Both phones connect to the Mac mini over Tailscale.

---

## System Architecture

```
[iOS App — React Native/Expo]  ←── Tailscale ───→  [Mac mini — FastAPI Server]
         │                                                      │
         │  • Camera + ML Kit OCR                              │  • Coqui XTTS-v2 (voice clone)
         │  • Audiobook player (ExoPlayer/expo-av)             │  • Open Library / Gutenberg search
         │  • Voice library UI                                  │  • PDF download + chapter split
         │  • Chapter nav + resume position                     │  • Audio synthesis + caching
         │  • Local audio file storage                          │  • AltServer (auto-refresh IPA)
         └──────────────────────────────────────────────────────┘
```

**Mac mini also runs AltServer** — auto-renews the sideloaded IPA on both iPhones over the Tailscale tunnel, so the 7-day AltStore expiry renews silently.

---

## iOS App

### Screens

**1. Library (Home)**
- List of all books with cover art (fetched from Open Library), title, author
- Resume button per book — drops user back to exact position
- Tap any book → opens Player

**2. Add Book**
- Two entry points: "Scan Title" (camera) and "Upload PDF" (file picker)
- Scan flow:
  1. Camera opens → user frames the book title/cover
  2. Google ML Kit Text Recognition v2 reads the title on-device
  3. App sends title to Mac server `/search-book`
  4. Server returns match: cover art, description, PDF URL
  5. User confirms → server downloads PDF, splits into chapters
  6. If not found: shows "not found" message + offers Upload PDF fallback

**3. Player**
- Full-screen layout: cover art, title, author
- Scrollable chapter strip for navigation
- Controls: play/pause, previous chapter, next chapter, 30-second rewind
- Speed selector: row of 5 buttons — `0.75×` `1×` `1.25×` `1.5×` `2×`
- Voice selector: dropdown showing current cloned voice
- Progress auto-saves to local storage every 10 seconds
- iOS lock screen controls (play/pause, chapter skip) via expo-av audio session

**4. Voices**
- List of cloned voice profiles (name + short sample playback button)
- "Add Voice" → file picker → selects MP3/MP4 from phone storage → uploads to Mac server → cloned in 30–90 sec → appears in list
- User can rename any voice (e.g. "Snoop", "Gwyneth")
- Optional: macOS system voice listed as "Narrator (System)" for high-quality non-cloned playback

**5. Settings**
- One input field: Mac mini Tailscale address (e.g. `100.x.x.x`)
- Connection status indicator (green = reachable, red = offline)

### Background Playback (iOS)

`expo-av` audio session set to `playback` mode. iOS treats the app identically to Spotify:
- Plays behind any other app
- Lock screen shows controls
- Survives screen-off
- Progress checkpoint written every 10 seconds; worst-case resume loss on force-quit is 10 seconds

### Tech Stack

| Component | Technology |
|---|---|
| Framework | React Native (Expo SDK) |
| OCR | Google ML Kit Text Recognition v2 (on-device) |
| Audio player | expo-av |
| Local storage | AsyncStorage (progress, voice list, server address) |
| Networking | Fetch API over Tailscale |
| Build / sideload | Expo EAS Build → `.ipa` → AltStore |

---

## Mac Server

### Runtime

- Python 3.11+
- FastAPI + Uvicorn
- Coqui TTS (xtts-v2 model)
- ffmpeg (speed adjustment, audio processing)
- pypdf (PDF text extraction + chapter splitting — reuses pdf-to-audio skill logic)

### Endpoints

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/health` | Ping — returns `{ status: "ok" }` |
| `POST` | `/search-book` | Title → book metadata + PDF URL |
| `POST` | `/download-book` | PDF URL → download, split chapters, return chapter list |
| `POST` | `/clone-voice` | Audio file upload → voice profile, returns `voice_id` |
| `POST` | `/synthesize` | Text + voice_id + speed → streamed MP3 audio |
| `GET` | `/voices` | List all cloned voice profiles |

### `/search-book` detail

1. Queries Open Library Search API (`openlibrary.org/search.json`)
2. Falls back to Project Gutenberg search if not found on Open Library
3. Returns: `{ title, author, cover_url, description, pdf_url, source }` or `{ found: false }`

### `/synthesize` detail

- Generates audio with Coqui XTTS-v2 using stored voice fingerprint
- Applies speed via `ffmpeg -filter:a "atempo=<speed>"` post-synthesis
- Streams MP3 in chunks — playback starts before full chapter is done
- **Caches output**: `<book_id>/<chapter_index>/<voice_id>/<speed>.mp3` — same request never re-generates
- Once a full book is cached, Mac does not need to be on for playback

### Voice Cloning

- Reference clip requirements: 5–30 sec clean speech, MP3 or MP4, no background music
- XTTS-v2 processes clip once, saves voice fingerprint (~2MB) to `voices/<voice_id>/`
- Quality improves with longer clips; 20–30 sec recommended for celebrity voices
- Speed is applied post-synthesis via ffmpeg — voice quality unaffected

### AltServer

Runs alongside FastAPI on Mac mini. Auto-renews IPA on both iPhones over Tailscale when they come online. No action required from either user after initial AltStore setup.

---

## Data Flow: First Book

```
1. User → camera → book title
2. ML Kit → OCR text → app
3. App → POST /search-book → Mac server
4. Mac → Open Library API → PDF URL
5. App shows match → user confirms
6. App → POST /download-book → Mac
7. Mac → downloads PDF → splits chapters → returns chapter list
8. App stores chapter list locally
9. User selects voice → taps Play
10. App → POST /synthesize (chapter 1) → Mac
11. Mac → XTTS-v2 → ffmpeg → streams MP3
12. App → expo-av plays stream → progress saved every 10s
```

---

## Distribution

| User | Setup |
|---|---|
| Owner | Install AltStore, sideload IPA, install Tailscale, join owner's network |
| Friend | Same as above — owner sends Tailscale invite link + IPA file |
| IPA renewal | Automatic via AltServer on Mac mini over Tailscale |

**Build command:** `eas build --platform ios --profile preview`
Output: `.ipa` file shared via iMessage / AirDrop / Google Drive.

---

## Build Phases

**Phase 1 — Core app + server skeleton**
- FastAPI server with `/health`, `/search-book`, `/download-book`
- App: Library, Add Book (scan + upload), basic player (Edge TTS voice, no cloning yet)
- Confirms the full pipeline works end-to-end before adding voice cloning

**Phase 2 — Voice cloning**
- Add `/clone-voice` and `/synthesize` with XTTS-v2
- Add Voices screen to app
- Replace Edge TTS placeholder with cloned voices

**Phase 3 — Full player + distribution**
- Background playback, lock screen controls, resume position
- Speed control
- AltStore IPA build + AltServer setup on Mac mini
- Tailscale setup guide for friend

---

## Open Questions / Constraints

- Mac mini must be awake for new audio generation; already-cached chapters play offline
- Reference clips for celebrity voices must be sourced by the user (YouTube interview clips, etc.)
- Open Library / Gutenberg coverage is limited to public domain books; modern titles won't be found
- AltStore free tier requires re-sign every 7 days if AltServer is unreachable
