# Wingram

**A real PC Instagram client written entirely in [Still](https://github.com/StillAdum/Still)** — no Electron, no Tauri, no HTML. The whole app is Still source (`.strc`) compiled to native Windows code through Still's C emitter. The one exception is the login: an embedded Edge (WebView2) window that signs you in to Instagram for real, so the feed behind it is **your actual feed**.

![Wingram](assets/banner.png)

## What it is

Wingram looks like instagram.com (dark web layout: top bar, left nav rail,
centered feed column) and implements the mobile-exclusive features the web
client never got — **notes with songs** front and center — plus a full
**Mods** system driven by plain-text `.strc` files.

### Features (v0.2 — all real data)

| Area | What you get |
| --- | --- |
| **Login** | Embedded WebView2 window on instagram.com/accounts/login — sign in once (2FA/checkpoints work), the app captures `sessionid`/`ds_user_id` automatically. Session persists in `config.ini`. Fallback: paste the sessionid cookie |
| **Feed** | Your real Instagram timeline: real post images (progressively downloaded to an on-disk cache), real like/comment counts, captions, timestamps; video posts marked with a play badge |
| **Notes + songs** | Compose 60-char notes like mobile, attach a song from a searchable picker, hear a **synthesized audio preview**, see the music sticker on the feed tray; notes persist in `wingram_notes.json` |
| **Profile** | Real header (avatar, bio, follower/following counts) for you and anyone you tap; recent-post grid |
| **Search** | Real user search via the same topsearch endpoint instagram.com uses |
| **Notifications** | Real activity inbox (likes/comments/follows), best-effort |
| **Reels** | Video posts from your timeline with poster frames + permalinks |
| **Messages** | Honest placeholder — Instagram's DM endpoints are GraphQL-only; no fake chats |
| **Settings → Mods** | List `<modname>.strc` files, toggle them live, hot-reload, persisted in `config.ini`; log out |
| **Backends** | **instagram** (default, sessionid web API) • fediverse live timelines (Mastodon/Pixelfed) |

### Songs in notes — how it works on PC

The mobile app lets you attach a song sticker to a note. Wingram does the
same on desktop: the composer has a **pick a song** button that opens a
searchable catalog, every song carries an original melody spec
(`"freq:ms,..."`), and previews are **synthesized at runtime** into a WAV in
memory and played through winmm — no audio assets, works on every machine.

## The login, step by step

1. Download the release **zip** (keep `wingram.exe` and `WebView2Loader.dll`
   together — the login webview loads the loader DLL from its own folder).
2. Run `wingram.exe`. The login card appears.
3. Click **Log in with Instagram** — an embedded Edge window opens on
   instagram.com/accounts/login. Sign in normally (2FA and "remember me"
   work). The runtime polls the browser's cookies in the background; the
   moment `sessionid` + `ds_user_id` exist, the window closes itself.
4. Your real feed loads. Next launch skips straight to it.
5. No WebView2 on the machine? The login card shows a **sessionid paste**
   fallback instead (devtools → Application → Cookies → `sessionid`).

**Unofficial-client warning:** Instagram offers no third-party API (the
Basic Display API shut down in December 2024). Wingram talks to the same
web API endpoints your browser does, with your session cookie. That is the
same category of client as every other sessionid-based Instagram app: it
violates Instagram's ToS, heavy automated use can get an account
rate-limited, and endpoints can drift without notice. It is read-mostly by
design (feed, profiles, search, notifications). Use an account you can
afford to risk.

## Build

Wingram needs the **Still 0.6.7+** compiler (gui/net/audio/media/json/webview/io stdlib modules; 0.6.5 added the WebView2 login backend, per-monitor DPI and windows-subsystem builds; 0.6.6 added `io.make_dir`):

```powershell
# get the compiler (Windows x64)
# download still-vX.Y.Z-windows-x64.zip from https://github.com/StillAdum/Still/releases
powershell -ExecutionPolicy Bypass -File .\install.ps1
still setup-cc          # portable TinyCC, one time

still check wingram.strc            # typecheck
still run wingram.strc              # dev run
still build wingram.strc -o wingram.exe --release
.\wingram.exe                       # the client
.\wingram.exe --selftest            # headless logic checks (CI uses this)
```

GitHub Actions does all of this automatically on every push — see
`.github/workflows/build.yml` — and attaches `wingram.exe`,
`WebView2Loader.dll` and a ready-to-run zip to every `v*` tag.

## Mods

Drop `.strc` files into the `mods/` folder next to the exe, then toggle them
in **Settings → Mods** (hot-reload button included). Format and a full
reference in [MODS.md](MODS.md). Quick taste:

```ini
# mods/my_theme.strc
[mod]
id = my-theme
name = My Theme
version = 1.0.0
author = you
description = Violet accent, deeper black.

[theme]
accent = 0x8B5CF6
bg = 0x050507

[features]
feed_density = compact
```

## Project layout

```
wingram.strc              entry point: window, frame loop, router, data pump, selftest
src/
  theme.strc              palette + color parsing
  types.strc              data models (User/Post/Note/Notif/...) + time/session helpers
  app.strc                the App state struct every screen renders from
  ui_kit.strc             widgets: buttons, inputs, icons, nav rail, toggles
  imagepipe.strc          progressive image downloads (cache/ + per-frame queue)
  songs.strc              song catalog + melody specs + preview playback
  mods_engine.strc        .strc mod scanning/parsing/applying + config.ini
  storage.strc            notes persistence (JSON) + HTML stripping
  adapters.strc           instagram web API client + fediverse backend
  screens/                login, feed, explore, reels, notes, dm, profile, notifications, settings
.github/workflows/        CI: typecheck → build → selftest → smoke → artifact/release
```

## Changelog

- **v0.2.0** — the "actually functioning app" release. Real Instagram login
  through an embedded WebView2 window (cookie capture included), the real
  home timeline with real images/counts, real profiles, real search, real
  notifications; per-monitor-v2 DPI scaling everywhere (crisp at 125/150%,
  layouts recompute from the live client size); windows are resizable and
  maximizable again; no terminal window behind the exe (windows-subsystem
  build); demo mode removed — every pixel of content comes from your
  account or the fediverse.
- **v0.1.2** — fix the crash-on-launch white-window bug (root cause was in
  the Still 0.6.3 compiler: `d.get(key, x.f)` with a field fallback could
  free the fallback before storing it; rebuilt against Still 0.6.4). CI now
  also opens a real window for 90 frames (`--smoke`) so GUI regressions
  fail the build. Fixes corrupted `[mod]` markers in docs and sample mods.
- **v0.1.1** — release packaging fixes (source zip, release notes).
- **v0.1.0** — first public version.

## Screenshots

Run it — or check the releases. The UI is fully drawn each frame with GDI
(rects, rounded rects, circles, text) through Still's `gui` module; post
images and avatars go through GDI+. There is not a single line of HTML or
JavaScript in this repository — the login window is the OS webview, the app
itself is 100% Still.

## License

Same as the Still repository's licensing (ask before redistributing).
