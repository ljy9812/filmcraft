# FilmCraft for HarmonyOS (OpenHarmony) — ArkWeb shell

Runs the official `filmcraft-web` build — the same static site published as
`filmcraft-web-<version>.zip` on the GitHub release — inside a HarmonyOS app. An
ArkTS shell wraps an ArkWeb `Web` component and serves the site from
the HAP's own `rawfile`: no network origin, no server-side code.

This branch (`ohos-webshell`) is the integrated form: the ArkWeb shell
packaging under `packaging/ohos/` plus the wasm-side CJK font change
merged from the `cjk-web-face` branch (which remains the clean
single-purpose form for an upstream PR). Build either variant from this
one checkout.

## How the shell works

| Concern | Mechanism |
|---|---|
| Serving | `onInterceptRequest` maps the virtual origin `https://local.filmcraft/` to `entry/src/main/resources/rawfile/web/` |
| Opening files | `onShowFileSelector` → system `DocumentViewPicker`; the picked URI is handed back to the page's `<input type=file>` |
| Saving | `a[download]` clicks — attached (document listener) and programmatic/detached (`HTMLAnchorElement.prototype.click` override) — are intercepted by a hook installed in `onPageEnd`; the blob is fetched, base64-chunked over a `MessageChannel` port to ArkTS, and written through `DocumentSavePicker` (with a filename field) |
| External links | dual path: `onLoadIntercept` for main-frame http(s) navigations + `multiWindowAccess(true)`/`onWindowNew` for `window.open` popups — both route to the system browser |
| Import (FS-Access) | the wasm's `showOpenFilePicker` path is neutralized at page start — ArkWeb implements the API and the wasm probes it with `Reflect.has`, so the properties are deleted outright (own + `Window.prototype`), routing imports to the system picker |

## Build (Windows host, DevEco Studio toolchain)

Prerequisites: DevEco Studio (for `hvigorw`, `ohpm`, and the HarmonyOS SDK),
`hdc` on PATH, and `gh` (GitHub CLI) for the release download.

```powershell
cd packaging\ohos
.\scripts\fetch-web.ps1     # deploy the web build into rawfile
.\scripts\build-hap.ps1     # build the Release HAP and install to the connected device
```

### CJK (Chinese) variant — in this branch

`crates/text/build.rs` whitelists the Hans interface face for wasm builds — with CRAFT_FONTS_DIR set, the zh UI renders natively.

Self-build recipe (Windows host, Rust stable, `trunk` 0.21.x; pin versions
per the upstream repo — on networks where the wasm-opt download hangs,
build with the optimizer off or assemble the .stage outputs by hand):

```powershell
git clone -b hans-wqy-microhei https://github.com/ljy9812/craft-fonts.git craft-fonts-cjk
$env:CRAFT_FONTS_DIR = "<repo>\craft-fonts-cjk"      # absolute path
cd apps\filmcraft-web
trunk build --release
# deploy dist\web\ into packaging\ohos\entry\src\main\resources\rawfile\web\:
# swap the wasm+glue pair, copy snippets\ and fonts\ when the build produced
# them, and rewrite index.html's asset hash — a stale hash (or a stale SRI
# integrity attribute) double-fetches the wasm behind a white window.
```

### Signing

The committed tree is deliberately signing-free (`"signingConfigs": []` in
`build-profile.json5`). To build a signed HAP, put your own signing fragment
at `packaging\ohos\local\signing.json` (gitignored) — one `signingConfigs`
entry as DevEco Studio writes it. `build-hap.ps1` injects it for the build
and restores the pristine file afterwards. Adjust `bundleName` in
`AppScope/app.json5` (currently the `com.example.filmcraft.web` placeholder)
to match your signing material.

## Verified status and known limitations

Verified on a HarmonyOS laptop (HAD-W24, API 23, physical keyboard +
touchscreen): install & launch, zh-Hans UI rendering (embedded Hans face),
media import through the system picker (the neutralized FS-Access open
path), project export round-trip, and text input via the physical keyboard.

Known limitations:

- Large-video memory pressure through the 256 KiB chunked port bridge and
  playback timing criteria not yet run.
