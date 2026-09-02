# Windows REAPER exact-artifact strategy

This note supersedes the Windows resource-only assumption in [Exact-artifact REAPER isolation](reaper-exact-artifact-isolation.md).

## Decision

Use a **version-pinned, user-prepared portable REAPER template**, copied to a fresh directory for every PEST run. On the template's first launch, its owner selects **No** when REAPER asks whether the portable install should scan system VST/CLAP/LV2 paths. PEST then sets only the scenario staging root, clears the scenario's scan caches, leaves `UserPlugins/FX` empty, clears `CLAP_PATH`, scans, and fails unless the reported inventory and instantiated identities match the manifest.

This is the smallest supported nondestructive Windows contract. It does not require a dedicated Windows user or VM and never edits standard plug-in roots. Cockos documents portable installs as independent installations with their own settings ([REAPER 7.79 User Guide, sections 1.3 and 12.25](https://www.reaper.fm/userguide/ReaperUserGuide779a.pdf)), and REAPER 7.50 introduced the portable-first-run system-path question; 7.61 and 7.68 fixed rescan and path-restoration defects after opting out ([official change log](https://www.reaper.fm/whatsnew.txt)).

The owner creates the portable install through Cockos's installer and handles the license outside PEST. PEST may copy that prepared directory unattended; it must not download, redistribute, install, license, or silently derive a portable template from a developer profile.

## Native poisoned-duplicate proof

On 2026-09-02, an authorized Windows 11 x64 VM ran REAPER `7.79/x64` from a disposable portable resource. The executable SHA-256 was `9fc03b563f7a9262167a71f32ebaea9f9cb29ae1cea44b255d7f6cddffab7b0f`.

The experiment did not run the installer or accept a license. It copied the VM's already installed 7.79 program files into scratch and created a fresh valid portable configuration; REAPER's own resource APIs then confirmed that it was using the scratch directory. Because constructing that marker is not documented on Windows, it is evidence for runtime behavior, not permission for PEST to synthesize templates. The product contract still requires installer-created portable input from the user or CI image owner.

Machine-wide poison copies remained installed and untouched in `C:\Program Files\Common Files\VST3` and `C:\Program Files\Common Files\CLAP`: each payload was 56,501,248 bytes with SHA-256 `db5cf18bace13eac970e3abf8f921c256a47694286db6fe8b5931e4848a7329c`. The declared staged VST3 and CLAP payloads were each 56,517,120 bytes with SHA-256 `e407838a3d79206d4c9037f72fecfc9446c76054f2914420c9ae7543ea0d68b0`.

After the first-run **No** choice, the portable resource persisted empty external roots:

```ini
vstpath64=
clap_path_win64=
lv2path_win64=;
```

The probe then set `vstpath64` and `clap_path_win64` to the stage, removed `reaper-vstplugins64.ini`, `reaper-vstshells64.ini`, and `reaper-clap-win64.ini`, and launched:

```text
reaper.exe -newinst -new -nosplash <probe.lua>
```

`GetResourcePath()` and `get_ini_file()` both resolved inside the disposable portable directory. The rebuilt caches contained REAPER's bundled Cockos VST2 effects, the staged `BUFFR.vst3`, and the staged `BUFFR.clap`; neither installed poison appeared. REAPER enumerated and instantiated:

```text
VST3  C:/Tools/PEST/results/windows-strategy-portable/stage/BUFFR.vst3/Contents/x86_64-win/BUFFR.vst3
      fx_type=VST3
      fx_ident=<same staged path><1865955641{42756666724470634D75736963303031

CLAP  com.derpcat.buffr
      fx_type=CLAP
      fx_ident=com.derpcat.buffr
```

A second run copied the prepared portable directory to a new location, restored the post-opt-out portable INI, cleared its caches, and completed unattended. It reproduced the same staged identities and hashes while the machine-wide poison remained installed. Both runs saved their project before quitting; teardown left no REAPER process. Native run evidence remains on the authorized VM under:

```text
C:\Tools\PEST\results\windows-strategy-portable
C:\Tools\PEST\results\windows-strategy-portable-clone
```

The identity fields used in the proof are public REAPER APIs: `EnumInstalledFX`, `GetResourcePath`, `get_ini_file`, and `TrackFX_GetNamedConfigParm`'s `fx_type`/`fx_ident` values ([ReaScript API](https://www.reaper.fm/sdk/reascript/reascripthelp.html)).

## Runner contract

For each exact REAPER build, PEST requires a prepared portable template whose first-run opt-out has already been made. For every scenario it must:

1. copy the template to a new writable directory;
2. verify executable hash/version and that both resource and INI paths are inside that directory;
3. stage only the declared artifacts, set only that root in `vstpath64` and `clap_path_win64`, clear `CLAP_PATH`, clear the three scan caches, and keep `UserPlugins/FX` empty;
4. scan and record the complete VST3/CLAP inventory;
5. instantiate by type-specific identity, not display name, and verify `fx_type` plus `fx_ident`;
6. fail closed on any undeclared entry, path, hash, modal, timeout, or surviving owned process.

PEST must not synthesize undocumented first-run keys. There is no documented CLI switch for the opt-out, and [the resource-only native proof](https://github.com/Matari-Audio/PEST/issues/18) showed that `-cfgfile` alone restores Windows' standard VST3 roots and lets an installed duplicate win. The supported unattended input is therefore the prepared portable template itself, not a copied user profile or a guessed INI fragment.

A dedicated user is not an isolation boundary because Steinberg defines a machine-wide VST3 root under `Program Files\Common Files\VST3` in addition to the per-user root ([VST3 plug-in locations](https://steinbergmedia.github.io/vst3_dev_portal/pages/Technical%2BDocumentation/Locations%2BFormat/Plugin%2BLocations.html)). A disposable VM remains an optional stronger containment backend, but it is not required for exact discovery once this portable-template precondition passes.
