![greymarch-online-linux](.github/preview.png)

# Greymarch Online on Linux

Unofficial notes for running the [Greymarch Online](https://greymarchonline.com/) launcher on Linux. This is not a Greymarch release, and it does not ship the game, the launcher, SKSE, or any mods.

This is the path that cleared the launcher mod check on this machine. Vortex did not deploy the collection. The badge cleared when every probed file in `required-mods.json` was present in the isolated `Data/` tree and its sha256 matched. A local window is not a connected session. Judge the game server with the status endpoint below. Do not treat a Vortex window as the install.

Tested against Skyrim Special Edition `1.6.1170.0`, SKSE `2.2.6`, launcher `0.1.42` (`latest.yml` names `Greymarch-Launcher-Setup-0.1.42.exe`, 115435593 bytes), and `GE-Proton11-7`. A newer launcher can move files. Read `latest.yml` and the unpacked `requirements.json` before treating a pin below as current.

## What not to do

- Do not install into the Steam library copy of Skyrim. A second mod list that deploys there will overwrite it.
- Do not press Play on Skyrim in the Steam library. Steam has to be open as the bridge. The library Play button is the other tree.
- Do not launch the loader with a second `umu-run` of the unix path under `$HOME`. Proton stops with `unable to use parent for game drive`.
- Do not launch the loader with the host `wine` binary. That null-derefs in the SKSE hook.
- Do not run the launcher's NSIS setup wizard. Extract it.
- Do not print or commit `state.json`. The login token sits next to `skyrimDir`.
- Do not hand an `nxm://` link to a Vortex whose `gamePath` is the Steam library tree. The wrapper forwards to whatever instance is already up. That installs into the other mod list.
- Do not click **Get Mods** in the launcher. A missing-mod card is a file probe. Hash-copy the probed file, then Check Again.
- Do not click the open server list. Under Wine that popup flashes and does not take the mouse. Focus the dropdown, press Down until the other server is highlighted, then Enter.
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

`latest.yml` on that same host names `Greymarch-Launcher-Setup-0.1.42.exe` at 115435593 bytes. Do not click through the NSIS wizard. `7z x` the setup exe, then `7z x` the `$PLUGINSDIR/app-64.7z` inside it. Copy that payload into:

```text
$HOME/Games/umu/umu-greymarch/drive_c/Program Files/Greymarch Launcher
```

An in-app update does not replace that extracted copy. electron-updater's NSIS install lands at:

```text
drive_c/users/steamuser/AppData/Local/Programs/greymarch-launcher
```

The shortcut must launch that directory when `Greymarch Launcher.exe` is there. Otherwise "restart to update" relaunches the old Program Files binary and the ready banner never clears. Read `package.json` `version` out of each `app.asar` before saying which build is running. On this machine the AppData copy is `0.1.42`. The Program Files copy is the older extract. A new updater asar does not carry local patches.

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

`0.1.42` does not include this fallback. `fileVersion()` still returns null when PowerShell returns empty, and it does not read the PE string. The local patch was not reapplied after that update, and the version row was not re-checked on `0.1.42`. If the row is blank, reapply the fallback and repack. Do not treat a missing row as a Steam version mismatch.

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

That login was not completed, so Vortex never deployed the collection. Do not treat a Vortex window as the install. The isolated profile's last `gamePath` was still the Steam library tree. It was not started, and it was not used to install. Starting it while that path is the library tree deploys into the other mod list.

What filled the badge was a hash copy into the isolated `Data/` only. 233 of 233 probed files matched. The launcher then showed 129 required mods installed.

- Files whose sha256 already matched `required-mods.json` were copied from an existing 1.6.1170 `Data/` tree or from that profile's staging mods. Files that did not match were left behind. The other tree was not deployed into, and it did not gain `GreymarchCore.esp`.
- Archives already under `~/.config/Vortex/downloads/skyrimse` with an exact `source.fileSize` from the newest `collection.json` were extracted. The probed member was copied only when its sha256 matched the live JSON.
- Pins that were not on disk were not handed to the running Vortex. That process was the other profile. The logged-in Nexus files page was opened without `nmm=1`, **Slow download** was used, and the archive was saved outside Vortex. Then the same hash-copy. Do not click **Mod manager download**. That fires `nxm://` at whatever Vortex is already up.
- One pin, True Northern Crowns, did not start from the Slow download button. The logged-in page had to request the download URL and the browser saved the archive. Do not print that URL. The archive is a FOMOD with more than one `True Northern Crowns.esp`. The `Main` copy did not match. `Non Outfit Replace/True Northern Crowns.esp` did. Hash every copy of the probed name. Do not install the FOMOD default because it is the first file in the archive.

The pin list is `collection.json` inside the newest collection archive, not the first download batch in the log. An exact `source.fileSize` match means the archive is present. That check is not the launcher hash gate.

The Play button stays **Download mods** while any `required-mods.json` file is missing, hash-mismatched, or not starred in the prefix `Plugins.txt`:

```text
drive_c/users/steamuser/AppData/Local/Skyrim Special Edition/Plugins.txt
```

The badge counts environment failures, plus each mod whose state is not ok, plus warnings. It does not count files, and it does not move when archives land. Vortex extracts into staging and deploys only enabled mods, so a finished download can still leave the badge red.

If the archive is already on disk, hash-copy the probed member into the isolated `Data/` at the probed path, and only when the file's sha256 matches the live JSON. In `7z l -ba` the member path starts at column 53. The plugin is often under `DATA/`, `data/`, or a nested folder. A FaceGen or Voice directory named after the esp is not the plugin. Take `fileId` and `fileSize` from the newest collection archive's `collection.json` for that modId. Do not match pack names.

The launcher removes plugins it treats as conflicts. Do not copy those back from another install.

### Face morphs

The game check can show **1 to review** after the mod badge is green. The warning names loose `.tri` files in `Data/meshes/actors/character/character assets` that match `(male|female)head*.tri` or `*chargen*.tri`. On this tree that was seven files, including `EyesFemaleChargen.tri` and `FemaleHeadChargen.tri`. None of them are in `required-mods.json` or the client manifest.

The launcher's own text says to move them out of that folder if the character appearance crashes (`BSFaceGenModel` / `MaleHeadNord`). They were removed from the isolated tree only. Several were hardlinks shared with the other install. Those paths were unlinked, not renamed, so the other tree kept the file. A backup of the bytes sits beside the isolated game, not inside `Data/`.

### Update ready

**Update game files**, one file, about 2.8 MB, was `Data/Platform/Plugins/skymp5-client.js`. The copy on disk did not match `manifest.json`. It was replaced from `https://files.greymarchonline.com/files/{sha256}` only after the download hash matched. That is not **Get Mods**, and it is not a Vortex deploy.

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

That copies `scripts/greymarch-online` to `$HOME/.local/bin/greymarch-online`, writes `$HOME/.local/share/applications/greymarch-online.desktop` with an absolute `Exec` (desktop files do not expand `$HOME`), pulls the 16 through 256 seal frames out of the launcher exe the shortcut starts, and refreshes the menu cache. After an in-app update that exe is the AppData copy. Pulling the seal from the Program Files exe leaves the menu on the old mark. There is no Walker binary on this desktop. The app menu, and anything that reads desktop files, picks the entry up from that file. Do not use `logo.webp`, the site favicon, `steam_icon_489830`, or a user hicolor `index.theme`. Do not restart the shell for the icon.

The script starts the AppData launcher when `Greymarch Launcher.exe` is there, and falls back to the Program Files extract otherwise. A click while that exe is already running does nothing. A launch with neither `WAYLAND_DISPLAY` nor `DISPLAY` set leaves a Wine process and no window, and the next click then no-ops. The script sets both when they are empty.

Steam has to be running before Play. The shortcut is not a Steam library launch. `UMU_USE_STEAM=1` uses the Steam client as the bridge, app `489830`. If the client is down, the script starts `steam -silent` and waits. Leave that client open. Do not press Play on Skyrim in Steam. That starts the library tree, not this one.

The script refuses to start if the game link is missing or points at the Steam library tree. It runs `umu-run` from inside the launcher directory, with the bare exe name and `--no-sandbox`.

Log in through the launcher (Discord). That login is not the game server.

The server list is a native dropdown, and it only appears when the account may join more than one server. Under Wine the open list flashes and the click never lands. Focus the dropdown, press Down until the other server is highlighted, then Enter. Spam-clicking does not select it.

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
| Launcher | `0.1.42`, `latest.yml` size 115435593. In-app update lives under `AppData/Local/Programs/greymarch-launcher` |
| Collection | Nexus `agfyia` |
| Engine Fixes | `7.0.20`, AE / `1.6.1170` plugin, preloader in the game root |

## License

MIT. See [LICENSE](LICENSE). Copies distributed before this change stay Apache-2.0.
