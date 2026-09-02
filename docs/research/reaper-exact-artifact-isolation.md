# Exact-artifact REAPER isolation

## Decision

PEST can make native REAPER test only the supplied CLAP/VST3 artifacts, but a merely empty `reaper.ini` is not sufficient. Use a two-phase, version-pinned resource template and copy it into a fresh directory for every scenario:

1. The user or CI image installs REAPER and gives PEST the native executable path. PEST never redistributes or silently installs REAPER.
2. Once per exact REAPER build, launch a disposable bootstrap resource with `-newinst -cfgfile <absolute>/reaper.ini -nosplash <bootstrap.lua>`, then exit.
3. Normalize that template: set only the staging directory in the active VST and CLAP path keys, remove all VST/CLAP scan caches, and leave `UserPlugins/FX` empty.
4. For each scenario, copy the normalized template, stage exactly one declared artifact (or one declared set), clear `CLAP_PATH`, launch with that scenario's absolute `-cfgfile`, and force a scan.
5. Before testing, fail unless REAPER reports the expected executable/version/resource/INI and its complete VST3/CLAP inventory equals the manifest. Instantiate by the reported type-specific identity, then re-read `fx_type` and `fx_ident` from the instance.

This keeps the user's REAPER profile untouched and makes an installed duplicate unreachable. Hard-linking or copying the artifact into staging is preferable to a symlink: the run receipt can bind one staged byte/tree hash to what REAPER scanned.

## Native matrix and exact settings

As of 2026-09-02 the current release is REAPER 7.79. Cockos supplies Linux x86-64, Windows x64, and a macOS Universal binary containing native ARM64; these are the three PEST V1 runners ([downloads](https://www.reaper.fm/download.php)). The executable paths are installation choices, but the usual entry points are `REAPER/reaper`, `REAPER/reaper.exe`, and `REAPER.app/Contents/MacOS/REAPER`.

| Runner | VST path key | CLAP path key | Active discovery caches to delete before scan |
| --- | --- | --- | --- |
| Linux x86_64 | `vstpath` | `clap_path_linux-x86_64` | `reaper-vstplugins64.ini`, `reaper-vstshells64.ini`, `reaper-clap-linux-x86_64.ini` |
| Windows x64 | `vstpath64` | `clap_path_win64` | `reaper-vstplugins64.ini`, `reaper-vstshells64.ini`, `reaper-clap-win64.ini` |
| macOS Apple Silicon | `vstpath64` | `clap_path_macos-aarch64` | `reaper-vstplugins_arm64.ini`, `reaper-vstshells_arm64.ini`, `reaper-clap-macos-aarch64.ini` |

The keys and architecture-specific filenames above were inspected in Cockos's exact 7.79 native downloads ([Linux x86_64](https://www.reaper.fm/files/7.x/reaper779_linux_x86_64.tar.xz), [Windows x64](https://www.reaper.fm/files/7.x/reaper779_x64-install.exe), [macOS Universal](https://www.reaper.fm/files/7.x/reaper779_universal.dmg)). Treat them as versioned adapter data, not timeless constants: reject an unknown `GetAppVersion()` platform suffix instead of guessing filenames.

REAPER's built-in usage defines `-cfgfile file.ini` as a **full path** selecting an alternate resource directory; `-newinst` prevents attachment to a user's running instance. REAPER's guide separately states that a portable install has its own configuration/settings and can run independently of the main installation ([REAPER 7.79 User Guide, sections 1.3, 1.19, 12.25](https://www.reaper.fm/userguide/ReaperUserGuide779a.pdf)). The same guide documents that VST paths are ordered, recursively scanned, persisted, and rescanned, and that CLAP has its own path list and rescan control.

### Why bootstrap then normalize

A 7.79 Linux probe launched with a fresh alternate `reaper.ini` containing only a staging `vstpath` still appended the existing `~/.vst3` path and populated `reaper-vstplugins64.ini` from installed plug-ins. After rewriting `vstpath` to staging only and deleting that cache, a second launch on the same machine—while an installed BUFFR duplicate remained in `~/.vst3`—cached only staged `BUFFR.vst3` plus REAPER's bundled Cockos VST2 effects. The analogous explicit CLAP path produced `reaper-clap-linux-x86_64.ini` containing only staged `BUFFR.clap` and its `com.derpcat.buffr` identity.

That result agrees with the guide: first-run/default setup can add system paths, macOS normally scans its system and user Audio Plug-Ins directories, and Windows normally uses `C:\Program Files\Common Files\VST3` ([REAPER 7.79 User Guide, section 1.19](https://www.reaper.fm/userguide/ReaperUserGuide779a.pdf)). Therefore PEST must never trust first-run defaults or a copied user cache. Bootstrap is configuration generation only; no test result may be emitted until paths are normalized and caches rebuilt.

Also keep the scenario resource's `UserPlugins/FX` empty. REAPER 7 added automatic CLAP scanning there ([REAPER change log](https://www.reaper.fm/whatsnew.txt)), which is intentionally independent of the configured external CLAP path.

## Required preflight evidence

The bootstrap/test Lua should report:

- `GetAppVersion()` for the exact version and native architecture;
- `GetExePath()`, `GetResourcePath()`, and `get_ini_file()` to prove executable/config boundaries;
- every `EnumInstalledFX()` name and type-specific identifier;
- after insertion, `TrackFX_GetNamedConfigParm(..., "fx_type")` and `"fx_ident"`.

These are public REAPER APIs: `EnumInstalledFX` enumerates installed FX and returns each name and identity, while `fx_type` and `fx_ident` expose an instantiated plug-in's type and type-specific identifier ([REAPER ReaScript API](https://www.reaper.fm/sdk/reascript/reascripthelp.html#EnumInstalledFX), [named FX configuration](https://www.reaper.fm/sdk/reascript/reascripthelp.html#TrackFX_GetNamedConfigParm)). `GetResourcePath` and `get_ini_file` provide the two paths that must equal the scenario directory ([resource API](https://www.reaper.fm/sdk/reascript/reascripthelp.html#GetResourcePath), [INI API](https://www.reaper.fm/sdk/reascript/reascripthelp.html#get_ini_file)).

Pass only if the VST3/CLAP inventory exactly matches declared identities and formats, the active cache contains no undeclared VST3/CLAP entry, and the staged hash still matches the input artifact. Console display names and scan order are not proof: REAPER documents duplicate-name precedence, so identity plus scan-root exclusivity is required ([REAPER 7.79 User Guide, section 1.19](https://www.reaper.fm/userguide/ReaperUserGuide779a.pdf)).

## Installation and licensing boundary

Cockos's download page permits a fully functional 60-day evaluation without registration, but the bundled EULA limits that copy to evaluation and requires a purchased license after 60 days. A purchased license permits use on one computer at any given time, and Cockos says the same key may be installed on multiple computers only when REAPER is used on one at a time ([purchase terms](https://www.reaper.fm/purchase.php)). The EULA also prohibits redistributing REAPER ([7.79 EULA in the official Linux package](https://www.reaper.fm/files/7.x/reaper779_linux_x86_64.tar.xz)).

Therefore PEST records and validates a user-provided executable/version but does not download, bundle, or license REAPER. Long-lived or parallel CI must be licensed appropriately with Cockos; evaluation installs are not a perpetual CI mechanism. Any CI license material is provisioned as a secret into the isolated resource, excluded from logs/receipts/artifacts, and owned by the user—not PEST.

## Remaining acceptance requirement

The path/cache names are present in all three 7.79 binaries, and Linux isolation was exercised against a real installed duplicate. Before implementation is called cross-platform complete, repeat the same poisoned-duplicate acceptance on native Windows x64 and macOS ARM64 and preserve each receipt. Binary inspection proves the adapter inputs; it does not substitute for native launch/scan/teardown evidence.
