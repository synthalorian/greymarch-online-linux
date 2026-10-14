# Greymarch Online on Linux

Unofficial notes for running the [Greymarch Online](https://greymarchonline.com/) launcher on Linux. This is not a Greymarch release, and it does not ship the game, the launcher, SKSE, or any mods.

Tested against Skyrim Special Edition `1.6.1170.0`, SKSE `2.2.6`, launcher `0.1.20`, and `GE-Proton11-7`. A newer launcher can move files. Read `latest.yml` and the unpacked `requirements.json` before treating a pin below as current.

## What not to do

- Do not install into the Steam library copy of Skyrim. A second mod list that deploys there will overwrite it.
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

Prefix id is `umu-greymarch`. Create the prefix by running any short command through umu once, or let the launch script below create it on first start. Then extract the launcher. Do not click through the NSIS wizard.

The feed is `https://files.greymarchonline.com/launcher/latest.yml`. Download the `path` it names (this setup used `Greymarch-Launcher-Setup-0.1.20.exe`).

```bash
SETUP="$HOME/Downloads/Greymarch-Launcher-Setup-0.1.20.exe"
WORKDIR="$HOME/Downloads/greymarch-launcher"
mkdir -p "$WORKDIR"
7z x -y -o"$WORKDIR" "$SETUP"
7z x -y -o"$WORKDIR/payload" "$WORKDIR/\$PLUGINSDIR/app-64.7z"
```

Copy the payload into the prefix:

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

Refetch `https://files.greymarchonline.com/required-mods.json` before judging the mod badge. The server list changes without a collection revision bump.

## 5. Version row

The launcher's version check shells out to `powershell.exe`. Wine's builtin returns empty, so the row fails closed even with `WINEDLLOVERRIDES=powershell.exe=n`.

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

Use a second Vortex profile. Do not launch `/usr/bin/vortex` for this tree. That wrapper adds `--download` when given an argument, and a bare launch uses `~/.config/Vortex`.

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

The pin list is `collection.json` inside the collection archive, not the first download batch in the log. An exact `source.fileSize` match in that profile's download directory means the archive is present. That check is not the launcher hash gate.

The Play button stays **Download mods** while any `required-mods.json` file is missing, hash-mismatched, or not starred in the prefix `Plugins.txt`:

```text
drive_c/users/steamuser/AppData/Local/Skyrim Special Edition/Plugins.txt
```

The badge counts environment failures, plus each mod whose state is not ok, plus warnings. It does not count files, and it does not move when archives land. Vortex extracts into staging and deploys only enabled mods, so a finished download can still leave the badge red.

If the archive is already on disk, hash-copy the probed member into the isolated `Data/` at the probed path, and only when the file's sha256 matches the live JSON. In `7z l -ba` the member path starts at column 53. The plugin is often under `DATA/`, `data/`, or a nested folder. A FaceGen or Voice directory named after the esp is not the plugin. Take `fileId` and `fileSize` from the newest collection archive's `collection.json` for that modId. Do not match pack names.

The launcher removes plugins it treats as conflicts. Do not copy those back from another install.

### Engine Fixes

Skyrim Souls warns when SSE Engine Fixes is missing, or when Memory Manager and Global Time are off. The preloader alone is not the plugin.

From Nexus mod `17230`, Engine Fixes `7.0.20`:

- Part 1, the SKSE64 preloader: `d3dx9_42.dll` in the game root, next to `SkyrimSE.exe`. Not in `Data/`.
- Part 2, the main file: the FOMOD option **SSE v1.6.1170 (Anniversary Edition)**, plus the Required files, into `Data/SKSE/Plugins/`.

That is `EngineFixes.dll`, `EngineFixes.toml`, `EngineFixes_preload.txt`, and `EngineFixes_SNCT.ini`. The stock toml already has `bGlobalTime = true` and `bOverrideMemoryManager = true`. A running session will not pick this up. Restart.

SKSE on this prefix loaded plugins from `Data/SKSE/Plugins` (capital `SKSE`). A lowercase `skse/plugins` directory is not the one it used.

## 7. Launch

Install `scripts/greymarch-online` to `$HOME/.local/bin/greymarch-online` and mark it executable. It refuses to start if the game link is missing or points at the Steam library tree. It runs `umu-run` from inside the launcher directory, with the bare exe name and `--no-sandbox`.

The first Play can sit on **Skyrim is starting** for a while. That hang is the launcher spawn, not a dead install. Do not start a second loader beside it.

Optional desktop entry: `scripts/greymarch-online.desktop`. `Exec` must be an absolute path. Desktop files do not expand `$HOME`.

The seal is the PNG frames in the launcher exe resource section (`RT_ICON` group 3): 16, 32, 48, 64, 128, 256. Do not use `logo.webp`, the site favicon, or `steam_icon_489830`. Write each frame to:

```text
$HOME/.local/share/icons/hicolor/<size>x<size>/apps/greymarch-online.png
```

Do not add a user hicolor `index.theme`. Then:

```bash
update-desktop-database "$HOME/.local/share/applications"
```

Do not restart the shell for the icon.

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
| Launcher feed | `0.1.20` from `latest.yml` |
| Collection | Nexus `agfyia` |
| Engine Fixes | `7.0.20`, AE / `1.6.1170` plugin, preloader in the game root |
