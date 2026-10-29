# Facial assets and missing-mesh diagnosis

These are targeted procedures from an executed repair, not a general texture-conversion recipe. A green launcher check does not test every facial asset. The filesystem corrections were checked; a fresh in-world inspection of the affected beard and eye choices is still required before claiming a visual fix.

## Before changing anything

- Fully exit Skyrim and Vortex. Work only on the isolated Greymarch tree, never Steam's library copy.
- Set `DATA="$HOME/Games/greymarch/Skyrim Special Edition/Data"` (adjust to your isolated install). Discover the **winning owning staging mod** from `Data/vortex.deployment.json` and its `source` entries; do not assume a staging folder ID from another install.
- Back up affected files and deployment metadata outside `Data` and staging. Keep these backups private: deployment snapshots can contain local paths and instance identifiers. Do not publish launcher state, account data, tokens, or signed download URLs.
- Hardlinks share bytes. Unlink only the intended directory entry or atomically replace it with a separately written file; do not truncate a shared inode. Preserve existing files on collisions and investigate differing content.
- Inspect the exact path requested by the installed mesh or log. Do not replace whole `Data`, `textures`, `Textures`, or `meshes` trees with symlinks. Existing case branches may contain different mod assets.

## Mouth/chin: malformed loose normal

A pale chin/mouth patch accompanied by an EngineFixes failure naming `MouthHuman_N.dds` warrants inspection of the loose override. In the recorded repair, `textures/actors/character/mouth/mouthhuman_n.dds` was a malformed 512×512, 24-bit RGB, ten-mip DDS, **1,048,703 bytes** long. Its payload was one byte short. The stock member was confirmed in the actual directory index of `Skyrim - Textures0.bsa`.

1. Inspect the deployed DDS header, mip payload and fresh log. A matching filename or RGB flags alone do not establish corruption; do not remove a valid custom mouth normal.
2. Confirm that `Skyrim - Textures0.bsa` contains `textures/actors/character/mouth/mouthhuman_n.dds` with a BSA-aware archive reader. Leave the archive unchanged.
3. Back up the malformed loose DDS. Unlink its names from both isolated `Data` and the owning installed staging mod (Total Character Makeover in this repair), including equivalent case twins. Do not copy the same malformed file from the Steam tree.
4. Remove only that path's deployment ownership record while Vortex is closed. With the loose override absent, the stock BSA member can load.

Removing only the live file is temporary: deployment can restore it from staging. Removing the bad staged source and its record survives a normal redeploy, but reinstalling the original mod archive can restore it. Recheck after reinstalling/updating that mod.

## Beard: exact-case paths, unchanged texture bytes

The fresh beard color cards were already DXT5. **No beard conversion or repaint was performed.** Inspect texture strings in the actual installed beard `.nif` first; different meshes may request different paths. The recorded exact aliases were:

| Exact destination, relative to Data | Existing source, relative to Data |
| --- | --- |
| `textures/actors/character/beards/BeardShort07.DDS` | `textures/actors/character/beards/beardshort07.dds` |
| `textures/actors/character/beards/BeardShort07_n.dds` | `textures/actors/character/beards/beardshort07_n.dds` |
| `textures/actors/character/beards/BeardShort06.dds` | `textures/actors/character/beards/beardshort06.dds` |
| `textures/actors/character/beards/BeardShort06_n.dds` | `textures/actors/character/beards/beardshort06_n.dds` |

The `06` requests were verified in Hallgarth beard NIF strings; the `07` pair was the Beards of Power compatibility correction.

For each **missing** destination, add a same-inode hardlink to the winning source in both its owning staging mod and live `Data`. If a destination already exists, compare content: keep an identical file, and stop on a difference rather than overwriting it. Record the new exact aliases under the same owning mod in deployment metadata. Verify source/alias content and inode identity in both trees.

## Eyebrows: no conversion

The fresh male/female brow color cards were already DXT5 and were left unchanged. `Brows.esp` was absent. No old Hvergelmir custom paths, face-mesh edits, eyebrow-record edits, or eyebrow DDS conversions were applied. The clean reinstall removed the previous mixed files; this follow-up did not establish a separate eyebrow-specific patch. Do not turn that history into a blanket recommendation to convert brows, normals, eyes, or hair textures.

## Eyes: distinguish payload corruption from split case branches

EngineFixes reported rejected eye DDS loads, including a request spelled `data\TEXTURES\actors\Character\eyes\...`. That wording alone did **not** prove corrupt pixels or establish which physical path the game opened.

The eleven unique logged filenames were:

```text
eyeblind.dds
eyeblue.dds
eyebrown.dds
eyebrown_n.dds
eyebrown_sk.dds
eyegreen.dds
eyegrey.dds
eyehazel.dds
eyehazelbrown.dds
eyesiceblue.dds
eyeyellow.dds
```

Canonical and staged files matched, had valid DXT5 headers and complete 256×256 / nine-mip payloads of 87,536 bytes, and decoded independently with ImageMagick. No padding, conversion or header rewrite was justified.

Check these branches separately on a case-sensitive filesystem:

| Branch relative to Data | Recorded finding |
| --- | --- |
| `textures/actors/character/eyes` | Complete canonical eye tree |
| `textures/actors/Character/eyes` | Separate real directory with Eyes of Beauty additions, missing the logged vanilla choices |
| `Textures/actors/character` | Separate real branch containing FaceGenData, but no eyes subtree |

If your layout differs, diagnose it rather than forcing this one. For the demonstrated split:

1. Preserve existing directories and files. Enumerate the canonical eye directory's **direct files**, locate each winner's staging owner, and add missing exact filenames as hardlinks in the mixed-`Character` branch in both staging and `Data`. Existing files must match the winner; stop on differences. The repair added 66 such aliases, covering more choices than the logged failures.
2. Add only the following **absent subtrees**, after confirming their canonical targets exist. Targets below are relative to each link's parent; never replace an existing directory to force a link.

   ```text
   textures/actors/Character/eyes/greymarch
     -> ../../character/eyes/greymarch
   textures/actors/Character/eyes/HUE_Standalone_2
     -> ../../character/eyes/HUE_Standalone_2
   textures/actors/Character/eyes/hue_standalone_2
     -> ../../character/eyes/hue_standalone_2
   Textures/actors/character/eyes
     -> ../../../textures/actors/character/eyes
   ```

3. Add ownership records for staged file aliases. These subtree links are missing-subtree lookup compatibility links, not converted textures. Keep a local record of them and check them after deployment; do not assume Vortex owns or preserves untracked symlinks.
4. Verify every logged filename through all three eye branches against the canonical winner, then independently decode the DDS files. If the canonical file itself is invalid or differs from the winning staged source, stop: that is a different branch of the diagnosis, not permission to apply this alias recipe blindly.

## Staging and deployment persistence

The executed repair was performed offline, without starting Vortex. It removed the mouth record and added beard/eye file-alias records in `Data/vortex.deployment.json`, preserving the existing instance ID, target/staging paths and unrelated records. Each added entry used its exact `relPath`, the winning mod's `source`, empty `target`, and source mtime in microseconds (`st_mtime_ns // 1000`). `deploymentTime` was refreshed in milliseconds and the JSON replaced atomically.

Inspect your installed manifest schema before any offline edit; do not paste another installation's snapshot or invent an instance ID. Back up first, and keep Vortex closed throughout. The live file, staged source and ownership record must agree. A direct `Data` edit alone is not a persistent mod fix. After a normal deployment, repeat the path/content checks and verify the absent mouth override and required subtree links. A mod reinstall/update may require reapplying the targeted correction. Deployment persistence has not been proven merely by editing the manifest.

## Read-only checks and fresh-session validation

Examples after setting `DATA` above (requires Python 3; DDS decode requires ImageMagick). This checks one beard pair, not the whole install; repeat for all requested aliases and their staged counterparts:

```bash
python3 - "$DATA" <<'PY'
import pathlib, sys
root = pathlib.Path(sys.argv[1])
a = root / 'textures/actors/character/beards/beardshort07.dds'
b = root / 'textures/actors/character/beards/BeardShort07.DDS'
assert a.is_file() and b.is_file(), 'source or exact alias missing'
assert a.read_bytes() == b.read_bytes(), 'content conflict'
assert a.samefile(b), 'alias does not share the source inode'
print('BeardShort07.DDS: content and inode match')
PY
magick identify -regard-warnings "$DATA/textures/actors/character/eyes/eyeblue.dds"
```

A decoder pass is not a renderer test. Fully exit Skyrim, restart through the Greymarch launcher, revisit the same beard/eye choices and inspect the **new** EngineFixes log. Failed lookups and already-loaded head parts can remain cached in the previous session. An absence of eye errors does not prove all eye choices rendered correctly.

The recorded checks verified the targeted filesystem paths/content and eye decoding. Launcher probes passed at the mouth/beard stage, but later full-install checks had unrelated client/runtime-file drift. Use current server manifests rather than restoring an old snapshot to manufacture a pass. Report launcher readiness, asset integrity, post-deploy persistence and fresh in-world appearance as separate results. No rendered-eye pass is claimed here.

## Red triangles / missing meshes, including near Fort Kastav

Treat red missing-mesh markers separately from facial DDS problems. **No verified Fort Kastav red-marker fix is established here.** A location or screenshot alone does not identify the missing asset.

1. Identify the affected placed reference and its winning base record with an appropriate console/record inspection tool. Follow its model field to the exact requested `.nif` path (including capitalization); for generated or alternate models, follow the actual winning record instead of guessing from the nearest building.
2. Check that exact path under the isolated `Data/meshes` tree. Also inspect case variants such as `Meshes` and mixed-case subdirectories. A similarly named file elsewhere is not proof the requested path resolves. Check symlink targets, file type and access, not just a directory listing.
3. Check the owning mod's staging payload and the selected archive/FOMOD member. If it exists only in staging, investigate enabled state, deployment destination, conflicts and the winning source. If absent there, verify the pinned archive and installation choices; do not extract unrelated options over the whole tree.
4. If the mesh is archive-backed, inspect the relevant BSA's actual member index and confirm the archive is installed and loaded by the appropriate plugin/configuration. Absence of a loose `.nif` is not proof it is missing. Conversely, a loose conflicting override can mask an archive member.
5. Apply only the correction supported by that evidence: restore the required pinned member, resolve its winning deployment, or add a missing exact-case alias after checking collisions. Preserve staging/ownership and verify after deployment, then restart and revisit the same references.

Do not convert facial DDS files, reinstall the entire collection, or alias whole asset roots as an unverified red-triangle fix. Until the requested `.nif`, its source and the fresh in-world result are identified, report the diagnosis as pending.
