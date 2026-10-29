# WebDAV for umbrelOS

A community app-store package that puts **all of your umbrelOS files — Home,
external drives, and network shares — behind one standard WebDAV server**, with
a modern in-browser file manager. It is powered by
[copyparty](https://github.com/9001/copyparty), an actively maintained,
single-file file server with read-write WebDAV, per-folder permissions,
thumbnails, a control panel, and connection instructions built into the UI.

## What you get

- **One app, everything visible.** On install it serves all three umbrelOS
  storage roots at once:
  - `/Home` — your umbrelOS Home folder
  - `/External` — the external drives shown in the Files app
  - `/Network` — the network shares shown in the Files app
- **A real file manager in the browser** (same URL as the WebDAV endpoint):
  drag-and-drop upload with progress, create/rename/delete files and folders,
  move things between Home/drives/shares, video and photo thumbnails, download
  a folder as zip.
- **Settings built in.** The browser UI has a control panel with server info,
  theme selection, a `Connect` page with copy-paste instructions for macOS
  Finder, Windows Explorer, Linux, Android and rclone, plus a config viewer and
  live config reload for admins.
- **Reliable WebDAV folder creation.** `MKCOL` (create folder), `PUT`, `MOVE`,
  `DELETE`, `PROPFIND` and `LOCK` are all exercised against this exact
  configuration (see Verification below). Folder creation works from both the
  web UI and WebDAV clients, including unicode/spaces in names.
- **Robust client behavior.** `PROPFIND Depth: infinity` answers with the
  RFC 4918 `propfind-finite-depth` precondition instead of a cryptic error, and
  `dav-auth` avoids the Windows bug where anonymous read access prevents
  login for write operations.
- `amd64` + `arm64` support via the pinned multi-arch `copyparty/ac` image.

## Requirements

- umbrelOS **2.0 or newer** for the native folder-access picker. On older
  versions the app still runs with the default whole-storage mapping, but the
  per-app folder chooser is a 2.0 feature.
- 64-bit Umbrel (x86_64 or ARM64). The image is ~56 MB compressed.

## Install as a community app store

1. Put this repository on a public GitHub repository. Umbrel fetches community
   stores from a Git URL, so it must be publicly reachable.
2. In umbrelOS: **Settings → App Store → Community App Stores → Add**.
3. Enter the repository URL and add the store.
4. Open the new **WebDAV App Store**, then install **WebDAV**.

During install, umbrelOS shows the three storage roots it will expose. You can
narrow any of them to a specific folder right there.

## Choose what is shared

By default the app exposes everything umbreld exposes: Home, external drives,
and network shares. To share only one folder:

1. Right-click **WebDAV** on the umbrelOS home screen → **Settings**.
2. Open **Storage → Access to your folders**.
3. Narrow **Home**, **External drive**, or **Network share** to a specific
   folder (for example just `Home/Instagram backup`). Save — umbrelOS remounts
   it and restarts the app. Each storage root can pick a different folder.

Changing a selection never deletes data; it only changes what is mounted.

## Connect

Open the app from the umbrelOS home screen. The app URL is also the WebDAV
endpoint:

```text
http://umbrel.local:3919/
https://umbrel.local:3919/
```

Login with username `umbrel` (shown under **Settings → Default credentials**
on the app's card) and the per-install password. That's the same UI/login you
use for WebDAV clients; copyparty is configured with a single account so there
is no separate password to keep track of.

The app's built-in **control panel** (bottom-left of the file manager) has a
`connect` page that shows ready-to-use commands and steps for Windows, macOS,
Linux and Android, generated from the address you are actually connected with.

Examples:

- **macOS Finder:** Go → Connect to Server → `http://umbrel.local:3919/`.
- **Windows:** Map a network drive to `http://umbrel.local:3919/`. Windows may
  require the WebDAV Redirector feature; rclone is much faster.
- **rclone:** `rclone config` → WebDAV remote, URL
  `http://umbrel.local:3919/`, vendor `other`, username `umbrel`.
- **Android / iOS:** any WebDAV app (e.g. FolderSync, Documents) with the same
  URL and credentials.

Use the hostname or IP your device uses to reach Umbrel; `umbrel.local` is the
usual local name.

## Settings menu

Two places to configure, both working out of the box:

1. **umbrelOS side:** App Settings → Storage → Access to your folders (which
   folders are shared) and Settings → Default credentials (the password).
2. **In the app:** the copyparty control panel — `Connect` pages for every
   platform, server info and free-space, theme picker, and (for the admin
   account, which is the `umbrel` user) a config viewer with a live `reload
   cfg` button so you can tweak `webdav/data/config/copyparty.conf` and apply
   it without reinstalling.

## Security notes

- Umbrel proxy auth is disabled (`PROXY_AUTH_ADD: "false"`) because WebDAV
  clients cannot send Umbrel's login cookie; the app enforces its own password
  auth instead, including over Basic-auth (`dav-auth`).
- No anonymous access anywhere: every volume requires the `umbrel` account.
  Password brute-force uses copyparty's default ban rule (9 failures/hour →
  24h ban).
- `Basic Auth` is only encrypted under HTTPS — prefer `https://...:3919/` on
  untrusted networks.
- One account means one permission level: whoever has the credentials can
  read/write/move/delete in everything that is currently shared. Prefer the
  per-folder narrowing in app settings over sharing your whole Home.
- The container runs as uid `1000` (the umbrel user) and only reaches the
  network from the container side (`--ipa lan` rejects internet-sourced
  connections); no ports are published beyond the app proxy.
- Search engines are told `noindex, nofollow` (`no-robots`).
- App-owned state (config, thumbnail cache) lives in `${APP_DATA_DIR}/data`,
  never inside your files.

## Package layout

```text
umbrel-app-store.yml          # community store metadata (id: webdav)
webdav/
  umbrel-app.yml              # manifest: folderAccess slots (home/external/network), port 3919
  docker-compose.yml          # app_proxy + copyparty/ac, account passed from ${APP_PASSWORD}
  hooks/pre-start             # creates /external and /network host dirs if missing
  data/config/copyparty.conf  # server config (committed, survives app updates)
  data/roots/.gitkeep         # the /roots bind mount target (must be committed)
```

Details worth knowing if you modify the package:

- The account password is **not** in the config file. `docker-compose.yml`
  runs with `command: [-a, umbrel:${APP_PASSWORD}]`; umbrelOS manages
  `${APP_PASSWORD}`, so credential rotation and updates flow through naturally.
- The volume root is `/roots` inside the container, mapped as a bind mount
  from `${APP_DATA_DIR}/data/roots`. This matters: copyparty measures the
  filesystem the volume *root* sits on, and Docker's `overlay` container fs
  makes it disable write access as a ramdisk safety measure. A bind mount gives
  it the real backing fs (`wram` adds a belt-and-braces override).
- `hist: /cfg/hists/` keeps the thumbnail/index cache in app data so browsing
  your folders does not create dotfiles inside them.
- The `folderAccess` ids are `home`, `external`, `network`. Keep those ids
  stable; changing them makes umbrelOS drop the user's saved folder choices.
- Internal port is 3923 (copyparty's default); the external port stays 3919.
  If you change the manifest `port`, keep `app_proxy.APP_PORT: 3923` as is.

## Updating copyparty

1. Pick the new copyparty release tag
   (`copyparty/ac`, e.g. `1.20.26`).
2. Get the **multi-arch manifest-list** digest (not a per-architecture
   digest), e.g. via `docker buildx imagetools inspect copyparty/ac:<tag>`
   or the Docker Hub tag API; update the image tag and digest together in
   `docker-compose.yml`.
3. Bump the manifest `version` and `releaseNotes`.
4. Re-run the linter: `node lint-apps.mjs webdav --check-images --root .`
   in a checkout of getumbrel/umbrel-apps (`.tools/lint-apps.mjs`).

## Verification

Reproducible test with the exact committed configuration:

`copyparty-sfx.py v1.20.26` was run locally with this package's
`data/config/copyparty.conf` (paths remapped) plus `-a umbrel:<password>`,
the same way compose passes it. Results:

| Check | Result |
|---|---|
| login page (unauthenticated `GET /`) | `200` |
| `PROPFIND Depth: 1` on `/` lists Home/External/Network | `207` |
| unauthenticated `PROPFIND` | `401` |
| `MKCOL` (create folder via WebDAV) | `201` |
| `PUT` then `GET` round trip inside a new folder | `201/200` |
| `MOVE` within a volume, and `Home → External` cross-volume | `201` |
| `LOCK` with a valid lockinfo body | `200` |
| UI folder creation (`POST act=mkdir`, incl. unicode + dots) | `201` |
| `DELETE` folder | `200` |
| control panel (`?cp`) and connect page (`?hc`) authenticated | `200` |
| `PROPFIND Depth: infinity` | `403` + `propfind-finite-depth` (RFC 4918) |
| `--ipa lan` / healthcheck hostname reachable | loopback `200`, LAN `200` |

The manifest and compose file pass the upstream app linter including its
multi-arch image check: `No issues found.`

Not yet exercised on a real umbrelOS device: the install-time folder picker
flow, the `pre-start` hook, and the app lifecycle (healthcheck, restart,
folder-access remount). Do a device test before submitting upstream.
