# Greymarch Online on Linux

Unofficial notes for running the [Greymarch Online](https://greymarchonline.com/) launcher on Linux. This is not a Greymarch release, and it does not ship the game, the launcher, SKSE, or any mods.

This is the path that produced a local game window on this machine. It is not a finished Vortex collection install, and the game server was offline the whole time. A local window is not a connected session.

Tested against Skyrim Special Edition `1.6.1170.0`, SKSE `2.2.6`, `Greymarch-Launcher-Setup-latest.exe` (same byte size as the `0.1.20` file named in `latest.yml`), and `GE-Proton11-7`. A newer launcher can move files. Read `latest.yml` and the unpacked `requirements.json` before treating a pin below as current.

## What not to do

- Do not install into the Steam library copy of Skyrim. A second mod list that deploys there will overwrite it.
- Do not press Play on Skyrim in the Steam library. Steam has to be open as the bridge. The library Play button is the other tree.
- Do not launch the loader with a second `umu-run` of the unix path under `$HOME`. Proton stops with `unable to use parent for game drive`.
- Do not launch the loader with the host `wine` binary. That null-derefs in the SKSE hook.
- Do not run the launcher's NSIS setup wizard. Extract it.
- Do not print or commit `state.json`. The login token sits next to `skyrimDir`.
- A local game window is not a connected session. If the auth status says `online: false`, the server is down.

## What you need

- A Steam copy of Skyrim Special Edition (app `489830`) that has been downloaded at least once, so the depot cache exists.
- `umu-run`
- [GE-Proton 11-7](https://github.com/GloriousEggroll/proton-ge-custom/releases), unpacked at `$HOME/.local/share/Steam/compatibilitytools.d/GE-Proton11-7-x86_64`
- `7z`, `rsync`, and Node (for `@electron/asar`)
- Vortex from the AUR package, the `/opt/Vortex/vortex` binary. Not a wrapper that injects `--download`, and not another mod manager.
- A Nexus account, for the collection

## 1. Isolated game tree

Steam's depot cache is not the library install. On a normal Steam client it lives here:

```text
$HOME/.local/share/Steam/ubuntu12_32/steamapps/content/app_489830
```

Merge those depots into a new folder. Do not rsync onto `steamapps/common/Skyrim Special Edition`, and do not touch `appmanifest_489830.acf`.

```bash
C="$HOME/.local/share/Steam/ubuntu12_32/steamapps/content/app_489830"
GAME="$HOME/Games/greymarch/Skyrim Special Edition"
mkdir -p "$GAME"
rsync -a "$C/depot_489831/Data/" "$GAME/Data/"
rsync -a "$C/depot_489832/" "$GAME/"
cp -a "$C/depot_489833/SkyrimSE.exe" "$GAME/SkyrimSE.exe"
```

Done only when `SkyrimSE.exe` has the same md5 as `depot_489833/SkyrimSE.exe` and a different inode from the Steam library exe.

## 2. Prefix and launcher

Prefix id is `umu-greymarch`. Create it with `umu-run wineboot --init` under `GAMEID=umu-greymarch` and `UMU_ID=umu-greymarch`, `PROTONPATH` pointing at `GE-Proton11-7-x86_64`. Do not let the first launch invent the prefix.

Download the installer that was actually used:

```text
https://files.greymarchonline.com/launcher/Greymarch-Launcher-Setup-latest.exe
```

`latest.yml` on that same host named `Greymarch-Launcher-Setup-0.1.20.exe` at the same byte size (`115114532`). Do not click through the NSIS wizard. `7z x` the setup exe, then `7z x` the `$PLUGINSDIR/app-64.7z` inside it. Copy that payload into:

```text
$HOME/Games/umu/umu-greymarch/drive_c/Program Files/Greymarch Launcher
```

Symlink only this prefix's Steam game path at the isolated tree:

```text
drive_c/Program Files (x86)/Steam/steamapps/common/Skyrim Special Edition
  -> $HOME/Games/greymarch/Skyrim Special Edition
```

If that link resolves to the Steam library tree, do not launch.

Proton needs the Steam client libraries in this prefix's Steam root, not copied into another prefix:

```bash
STEAM="$HOME/.local/share/Steam"
ROOT="$HOME/Games/umu/umu-greymarch/drive_c/Program Files (x86)/Steam"
ln -sfn "$STEAM/steamclient64.dll" "$ROOT/steamclient64.dll"
ln -sfn "$STEAM/legacycompat/steamclient.dll" "$ROOT/steamclient.dll"
ln -sfn "$STEAM/legacycompat/Steam.dll" "$ROOT/Steam.dll"
```

Before the first launch, set `skyrimDir` in both of these to the Windows path of the isolated tree (`C:\Program Files (x86)\Steam\steamapps\common\Skyrim Special Edition` once the symlink above exists):

- `drive_c/GreymarchLauncher/state.json`
- the steamuser Roaming copy of that file

Do not cat that file into a terminal you share. The token is in it.

`SteamAppId` and `SteamGameId` stay `489830`.

## 3. SKSE

Read `config/requirements.json` from the unpacked asar. This setup pinned the official archive named there:

- url: `https://skse.silverlock.org/beta/skse64_2_02_06.7z`
- sha256: `d7297f1a1d613e5265e1af4dbbfe8bd37a32719c1ccef363fc6187fa6eba0848`
- loader: `skse64_loader.exe`
- dll: `skse64_1_6_1170.dll`

Copy the loader and the 1.6.1170 dll next to `SkyrimSE.exe`. Copy `Data/Scripts/*.pex` into the isolated `Data/Scripts/`.

Linux is case-sensitive. The archive ships lowercase script names. The launcher probes PascalCase (`Actor.pex`). Hardlink each probed name onto the lowercase file. Do not keep two copies.

## 4. Client files

```text
GET https://files.greymarchonline.com/manifest.json
GET https://files.greymarchonline.com/files/{sha256}
```

Write each `files[].path` under the isolated `Data/`. Replace a file only after its sha256 matches. A full match is what the launcher calls up to date.

`Data/Platform/Plugins/skymp5-client-settings.txt` was written in the isolated tree from the launcher config (server address, port, and the autoplay flag). The other tree's copy of that file was a different inode and was not overwritten. Do not publish the values.

Refetch `https://files.greymarchonline.com/required-mods.json` before judging the mod badge. The server list changes without a collection revision bump.

## 5. Version row

The launcher's version check shells out to `powershell.exe`. Wine's builtin returns empty.

A native `powershell.exe` built for this prefix, plus `powershell.exe=native` in the prefix registry and `WINEDLLOVERRIDES=powershell.exe=n`, did not clear the row. Under `umu-run` the launcher child never ran that shim. The fix that cleared it is a fallback in the launcher itself.

Unpack `resources/app.asar`, and in `src/main/core/skyrim.js` fall back to the PE version string when that shell returns null:

```js
async function fileVersion(file) {
  if (!exists(file)) return null;
  const out = await run(
    "powershell.exe",
    ["-NoProfile", "-NonInteractive", "-Command", "(Get-Item -LiteralPath $env:GM_FILE).VersionInfo.FileVersion"],
    { GM_FILE: file }
  );
  const fromShell = out ? out.trim() || null : null;
  return fromShell || readFileVersion(file);
}

function readFileVersion(file) {
  let data;
  try {
    data = fs.readFileSync(file);
  } catch {
    return null;
  }
  const key = Buffer.from("FileVersion", "utf16le");
  const at = data.indexOf(key);
  if (at < 0) return null;
  let i = at + key.length;
  while (i + 1 < data.length && data[i] === 0 && data[i + 1] === 0) i += 2;
  const chars = [];
  for (; i + 1 < data.length && chars.length < 64; i += 2) {
    const c = data.readUInt16LE(i);
    if (c === 0) break;
    chars.push(String.fromCharCode(c));
  }
  const ver = chars.join("").trim();
  return ver || null;
}
```

Repack the asar and restart the launcher. The row is clear only when the window shows `1.6.1170.0`. A future launcher that already reads the PE string does not need this.

```bash
npx --yes @electron/asar extract resources/app.asar /tmp/greymarch-asar
# edit src/main/core/skyrim.js
npx --yes @electron/asar pack /tmp/greymarch-asar resources/app.asar
```

## 6. Mods

The Play button stayed **Download mods** until every probed file in `required-mods.json` was present, hash-matched, and starred in the prefix `Plugins.txt`. Vortex did not do that install.

A separate Vortex window was opened with the command below. `--user-data-dir` alone is ignored and opens `~/.config/Vortex`. That first window was an existing profile with an Apply Changes dialog. It was killed. Apply was not clicked. The env form below is what actually opened a fresh login window:

Do not launch `/usr/bin/vortex` for this tree. That wrapper adds `--download` when given an argument, and a bare launch uses `~/.config/Vortex`.

```bash
env ELECTRON_USERDATA="$HOME/.config/Vortex-greymarch" \
  ELECTRON_APPDATA="$HOME/.config/Vortex-greymarch" \
  ELECTRON_OZONE_PLATFORM_HINT=x11 GDK_BACKEND=x11 \
  /opt/Vortex/vortex --ozone-platform=x11 \
  --user-data-dir="$HOME/.config/Vortex-greymarch"
```

A space after `--user-data-dir` is ignored. The profile is the right one only when `gamePath` in the staging-folder check is the isolated tree. A window title is not that check. If the log rotated, read `Data/vortex.deployment.json` and its `targetPath`. Do not use `hourly.json`.

Staging and downloads for this profile must sit on the same filesystem as the isolated game. Do not let discovery attach `skyrimse` to the Steam library path. Switching to an empty profile while `gamePath` is still the library install deploys that empty set into the library tree.

Collection:

```text
https://www.nexusmods.com/games/skyrimspecialedition/collections/agfyia
```

That login was not completed, so Vortex never deployed the collection. Do not treat a Vortex window as the install.

What filled the badge was a hash copy into the isolated `Data/` only:

- Files whose sha256 already matched `required-mods.json` were copied from an existing 1.6.1170 `Data/` tree. Files that did not match were left behind. The other tree was not deployed into, and it did not gain `GreymarchCore.esp`.
- The rest were already-downloaded archives under `~/.config/Vortex/downloads/skyrimse`. The probed member was extracted and copied only when its sha256 matched the live JSON.

The pin list is `collection.json` inside the collection archive, not the first download batch in the log. An exact `source.fileSize` match means the archive is present. That check is not the launcher hash gate.

The Play button stays **Download mods** while any `required-mods.json` file is missing, hash-mismatched, or not starred in the prefix `Plugins.txt`:

```text
drive_c/users/steamuser/AppData/Local/Skyrim Special Edition/Plugins.txt
```

The badge counts environment failures, plus each mod whose state is not ok, plus warnings. It does not count files, and it does not move when archives land. Vortex extracts into staging and deploys only enabled mods, so a finished download can still leave the badge red.

If the archive is already on disk, hash-copy the probed member into the isolated `Data/` at the probed path, and only when the file's sha256 matches the live JSON. In `7z l -ba` the member path starts at column 53. The plugin is often under `DATA/`, `data/`, or a nested folder. A FaceGen or Voice directory named after the esp is not the plugin. Take `fileId` and `fileSize` from the newest collection archive's `collection.json` for that modId. Do not match pack names.

The launcher removes plugins it treats as conflicts. Do not copy those back from another install.

### Engine Fixes

The game window came up before this. Skyrim Souls then warned that SSE Engine Fixes was missing, or that Memory Manager and Global Time were off. The preloader alone is not the plugin. On this machine the preloader `d3dx9_42.dll` in the game root already matched the Nexus file. The plugin did not.

From Nexus mod `17230`, Engine Fixes `7.0.20`:

- Part 1, the SKSE64 preloader: `d3dx9_42.dll` in the game root, next to `SkyrimSE.exe`. Not in `Data/`.
- Part 2, the main file: the FOMOD option **SSE v1.6.1170 (Anniversary Edition)**, plus the Required files, into `Data/SKSE/Plugins/`.

That is `EngineFixes.dll`, `EngineFixes.toml`, `EngineFixes_preload.txt`, and `EngineFixes_SNCT.ini`. The stock toml already has `bGlobalTime = true` and `bOverrideMemoryManager = true`. A running session will not pick this up. Restart.

SKSE on this prefix loaded plugins from `Data/SKSE/Plugins` (capital `SKSE`). A lowercase `skse/plugins` directory is not the one it used.

## 7. Shortcut

One command, after the prefix and the launcher payload are in place:

```bash
scripts/install-shortcut
```

That copies `scripts/greymarch-online` to `$HOME/.local/bin/greymarch-online`, writes `$HOME/.local/share/applications/greymarch-online.desktop` with an absolute `Exec` (desktop files do not expand `$HOME`), pulls the 16 through 256 seal frames out of `Greymarch Launcher.exe`, and refreshes the menu cache. There is no Walker binary on this desktop. The app menu, and anything that reads desktop files, picks the entry up from that file. Do not use `logo.webp`, the site favicon, `steam_icon_489830`, or a user hicolor `index.theme`. Do not restart the shell for the icon.

Steam has to be running before Play. The shortcut is not a Steam library launch. `UMU_USE_STEAM=1` uses the Steam client as the bridge, app `489830`. If the client is down, the script starts `steam -silent` and waits. Leave that client open. Do not press Play on Skyrim in Steam. That starts the library tree, not this one.

The script refuses to start if the game link is missing or points at the Steam library tree. It runs `umu-run` from inside the launcher directory, with the bare exe name and `--no-sandbox`.

Log in through the launcher (Discord). That login is not the game server.

A direct `umu-run` of `skse64_loader.exe` from the unix game path died in the Proton hook and was killed. Host `wine` is not a substitute. The launcher's own Play spawn is the launch that stayed up.

The first Play sat on **Skyrim is starting**, then the window came up. That hang was the launcher spawn, not a dead install. Do not start a second loader beside it.

## 8. Offline

The launcher asks `https://auth.greymarchonline.com/api/status`. `{"online":false}` means the game server is down. Nothing on the machine flips that. The window can still be a local boot.

## Pins from this setup

| Piece | Value |
| --- | --- |
| Game | Skyrim SE `1.6.1170.0`, app `489830` |
| SKSE | `2.2.6`, `skse64_1_6_1170.dll` |
| Proton | `GE-Proton11-7-x86_64` via `umu-run` |
| Prefix | `$HOME/Games/umu/umu-greymarch` |
| Game tree | `$HOME/Games/greymarch/Skyrim Special Edition` |
| Launcher | `Greymarch-Launcher-Setup-latest.exe` (115114532 bytes; `latest.yml` names that size as `0.1.20`) |
| Collection | Nexus `agfyia` |
| Engine Fixes | `7.0.20`, AE / `1.6.1170` plugin, preloader in the game root |

## License

MIT. See [LICENSE](LICENSE). Copies distributed before this change stay Apache-2.0.
