# Spotify OAuth Setup on Android/Termux

## Prerequisites

- A Spotify account (Free or Premium — playback control needs Premium)
- Hermes Agent with the bundled Spotify plugin (auto-loaded)

## Step 1: Create a Spotify Developer App

1. Go to **[Spotify Developer Dashboard](https://developer.spotify.com/dashboard)**
2. Click **Create App**
3. **App name:** anything (e.g. "Hermes Agent")
4. **App description:** optional
5. **Redirect URI:** `http://127.0.0.1:43827/spotify/callback`
   - This must be **exactly** this value — Hermes starts a callback server on 127.0.0.1:43827
6. **APIs:** Check "Web API"
7. Save → copy your **Client ID**

## Step 2: Save the Client ID

```bash
echo 'export HERMES_SPOTIFY_CLIENT_ID="your_client_id_here"' >> ~/.hermes/.env
```

Or set it directly in the env for the current session:
```bash
export HERMES_SPOTIFY_CLIENT_ID="your_client_id_here"
```

## Step 3: Run the Auth Flow

```bash
hermes auth spotify
```

On Termux/Android, the browser won't auto-open. The CLI prints an authorization URL.
Open that URL in your device browser, log in to Spotify, and authorize.

Hermes starts a local callback server on `http://127.0.0.1:43827/spotify/callback`.
When the browser redirects back, Hermes captures the code and exchanges it for tokens.

**If the callback fails** (e.g. mobile browser can't reach localhost):
- Use `hermes auth spotify --no-browser` to get just the URL
- Copy-paste the URL into a browser on the same device
- After authorizing, Spotify redirects to `http://127.0.0.1:43827/...` which hits the Hermes callback server

## Step 4: Verify

Check that credentials were saved:

```bash
cat ~/.hermes/auth.json | grep spotify
```

Or just ask the agent to verify — the tools are gated on `_check_spotify_available()`.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `spotify_client_id_missing` | Client ID not set. Check `HERMES_SPOTIFY_CLIENT_ID` in `~/.hermes/.env` |
| `redirect_uri_mismatch` | The redirect URI in your Spotify app settings doesn't match `http://127.0.0.1:43827/spotify/callback` |
| Auth succeeds but tools don't work | Run `hermes auth spotify` again — tokens may have expired |
| Browser can't reach localhost on callback | On Android, the same-device browser can reach 127.0.0.1. If using a separate device, SSH tunnel (see `hermes auth spotify --help` for loopback SSH hints) |

## The 7 Tools (once authorized)

| Tool | Emoji | What it does |
|------|-------|-------------|
| `spotify_playback` | 🎵 | play, pause, next, prev, seek, volume, shuffle, repeat, get state |
| `spotify_devices` | 🔈 | list available devices, transfer playback |
| `spotify_queue` | 📻 | get queue, add tracks |
| `spotify_search` | 🔎 | search catalog (tracks, albums, artists, playlists) |
| `spotify_playlists` | 📚 | list, get, create, add/remove items, update |
| `spotify_albums` | 💿 | get album info, tracks |
| `spotify_library` | ❤️ | save/remove/list saved tracks & albums |

Playback-mutating actions (play, pause, seek, volume) require Spotify Premium.
Search, library, and playlist operations work on Free.