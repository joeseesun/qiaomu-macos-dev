# Tauri Desktop Apps (macOS first, also Windows/Linux)

Use this file instead of the Swift generator when the app is Tauri 2 (Rust + WebView UI). The design taste, details checklist, signing identity, `notary` profile, GitHub and release rules of this skill still apply. Distilled from SubCast / 声渡 (github.com/joeseesun/subcast, Tauri 2 + React/TS + zustand + reqwest + rusqlite), shipped notarized for Mac (arm64 + x64) with CI builds for Windows and Linux.

## WebView runtime (WKWebView on Mac, WebView2 on Windows)

| Symptom | Cause | Fix |
|---|---|---|
| Unsubscribe / delete buttons "do nothing" | `window.confirm` shows no dialog in WKWebView and returns `false` | Never use `confirm` / `alert` / `prompt`. Ship one in-app confirm dialog: `ask({title, message, ok, danger}) → Promise<boolean>` plus a host component mounted once in `App` |
| Sidebar clicks sometimes ignored, worse while playing | Components subscribed to the whole store re-render on every playback tick; a component *defined inside* another component gets a new identity each render, so React remounts it and the click lands on a dead node | Select slices (`useStore(useShallow(s => …))`); never define components inside components (use plain render functions or hoist them). Verify with a MutationObserver: zero sidebar mutations during playback |
| YouTube embed refuses to play in the release build | YouTube rejects the `tauri://` origin | Serve the UI from `tauri-plugin-localhost` (`http://localhost:<fixed port>`). Add that origin to the IPC capability's remote URLs; a different port loads the page but every command is denied. A running older copy can hold the port, so quit it before testing the signed build |
| HLS stream plays on Mac, not on Windows | WebView2 has no native HLS | Lazy-load hls.js only when `canPlayType('application/vnd.apple.mpegurl')` is empty |
| Translated UI breaks logic | Code matched Chinese error text with `includes`/regex, or prompts were module-level constants frozen at import | Match on raw codes or a separate raw field; build prompts with functions called at use time; one i18n key per purpose (a button label and a heading need different keys in Japanese) |
| Menu shortcut (⌘,) dead in the packaged app | Shortcut only bound in JS | Put it on a native menu item and test in the packaged app, not `tauri dev` |

New Tauri commands need all three: the Rust `#[tauri::command]` + `invoke_handler`, the `build.rs` AppManifest command list, and `capabilities/default.json`. Missing one fails only at runtime.

## Rust networking

- reqwest "error sending request for url" on some hosts, especially behind a proxy: pooled idle connections were closed upstream and reused. Set `pool_idle_timeout(20s)` and `tcp_keepalive(30s)`, retry idempotent GETs twice with backoff, and turn errors into a human message with the innermost `source()` and the host (timeout vs connect vs other).
- Independent fetches for one page go in parallel (`tokio::join!`); sequential fetches made a channel page take 5–10 s, parallel ~1 s.
- Show cached data first, refresh in the background after a TTL, and give error states a 重试 button.
- Schema changes migrate on startup with a unit test.

## Auto-update (tauri-plugin-updater)

- **No delta updates**: every update is the full package (~7 MB Mac tarball, 10–20 MB installers). Say so to the user.
- Wire it end to end. SubCast once shipped a finished `UpdateBanner` and `startUpdateChecks()` that nothing called. Check: startup check (~10 s after launch) and every 6 h; phases idle → checking → available → downloading → ready → installing / error; default `autoUpdate: true` with a toggle in About; a dot on the settings button when an update is ready; "重启即可完成更新".
- Keys: `tauri signer generate`; private key in `~/.tauri/<app>-updater.key` (never in the repo) and in the GitHub secret `TAURI_SIGNING_PRIVATE_KEY`; only the pubkey goes in `tauri.conf.json`. Losing the key strands every install.
- `latest.json`: `version`, `notes`, `pub_date`, `platforms` (`darwin-aarch64`, `darwin-x86_64`, `windows-x86_64`, `linux-x86_64`, each `{url, signature}`) plus your own `downloads` map for the website. Upload it **last**, after every file it points to.
- Verify each platform's signature against the pubkey from `tauri.conf.json` (minisign: write the decoded pubkey to a file and use `-p`; passing the base64 inline fails with "base64 conversion failed").
- Test a real update: install the previous release, launch, wait for the banner, restart, check the version in About.

## Mac signing and notarization

`npm run tauri build -- --target <aarch64|x86_64>-apple-darwin --bundles app` with `APPLE_SIGNING_IDENTITY` set, per arch. Then:

1. `codesign --verify --deep --strict` the app; zip with `ditto -c -k --keepParent`; notarize the zip; staple the app.
2. **Rebuild the updater tarball after stapling** (`COPYFILE_DISABLE=1 tar -czf`), then sign it. Tauri's own tarball was made before stapling.
3. Build the DMG yourself (staging dir + `ln -s /Applications`), sign, notarize, staple, `spctl -a -t open --context context:primary-signature -v` → "Notarized Developer ID".

Script traps that aborted a release after Apple had accepted it:

- `notarytool … | tee | grep -q Accepted` under `set -o pipefail`: grep exits early, tee dies of SIGPIPE (141), the script stops. Capture output into a variable, print it, then `grep -q <<<"$out"`.
- `tauri signer sign -f <key>` refuses when `TAURI_SIGNING_PRIVATE_KEY` is also in the environment (the build exports it): run it under `env -u TAURI_SIGNING_PRIVATE_KEY -u TAURI_SIGNING_PRIVATE_KEY_PASSWORD`.
- Notarization needs the keychain unlocked; with the screen locked the `notary` profile can't be read. Poll `notarytool history` and continue after unlock.

## CI (Windows, Linux, fallback Mac)

- Windows and Linux build in GitHub Actions (`tauri-action`); Mac releases are built and notarized locally. **Do not let CI upload Mac assets to the release**: publishing the draft re-triggered CI, whose unsigned same-name DMG overwrote the notarized one. Mac CI keeps its output as an Actions artifact.
- `macos-13` runners are gone; use `macos-15-intel` for x64. Native crates (whisper-rs) on Windows need `LIBCLANG_PATH`; Linux needs CMake.
- Signing secrets go from a user-run script straight into `gh secret set`, never through the agent.
- Windows without Authenticode shows SmartScreen; say so in the notes ("更多信息 → 仍要运行"). The updater's minisign signature is not code signing.

## Windows runtime (learned from a user's "MSVCP140.dll was not found")

- **Any C++ dependency (whisper.cpp via whisper-rs, etc.) makes the exe import MSVCP140 / VCRUNTIME140**, which PCs without the Visual C++ Redistributable lack: the app dies before any UI. Link the runtime statically with `src-tauri/.cargo/config.toml`: `[target.x86_64-pc-windows-msvc] rustflags = ["-C", "target-feature=+crt-static"]` (the cmake crate follows it for native code).
- **Third-party executables you download at runtime have their own imports.** CrispASR's Windows build needs MSVCP140, VCRUNTIME140_1 and VCOMP140 (OpenMP). Ship the app-local redistributable DLLs as Windows-only resources (`tauri.windows.conf.json` → `bundle.resources`), copied in CI from the runner's `VC\Redist\MSVC\<ver>\x64\Microsoft.VC*.CRT` and `...OpenMP`, and copy them next to that executable before running it (the exe's folder is searched first).
- **Gate it in CI**: parse the PE import table of `target/release/<app>.exe` (a 30-line Python script, no Visual Studio tools needed) and fail on `msvcp140.dll`/`vcruntime140*.dll`/`vcomp140.dll`; list the NSIS installer with `7z l` and fail if the DLLs are missing. Check downloaded binaries the same way on the Mac before shipping a feature that runs them.
- `webviewInstallMode: { type: "embedBootstrapper" }` so the WebView2 install doesn't depend on downloading the bootstrapper.
- Shortcut labels: write them with ⌘ and replace with `Ctrl+` off macOS in one place (the i18n `t()` output); hide the macOS traffic-light spacer when not on a Mac (`:root:not(.is-mac)`).

## YouTube in a Tauri WebView

- Embeds need an http(s) origin (see `tauri-plugin-localhost` above). Only errors 101/150 mean the owner disallows embedding; 152/153 are player/referrer problems on your side; say which.
- The fallback is yt-dlp. Prepare it when a video page opens (first download is ~35 MB), show download progress and the step being waited on, cap the whole wait, and always offer "Open on YouTube". Mirror the binary on your own server with SHA-256 verification against the release's `SHA2-256SUMS` (users in mainland China often can't reach GitHub); keep the mirror outside the website root so site deploys can't delete it.
- WKWebView plays YouTube HLS natively. **WebView2 can't, and hls.js can't read YouTube's HLS because playlists and segments carry no CORS header** (`manifestLoadError`, code 0). A progressive MP4 (format 18) plays anywhere but is 360p. For HD: a loopback proxy on 127.0.0.1 with a random path token and a YouTube-hosts allowlist adds `Access-Control-Allow-Origin` and rewrites playlist URIs (entry lines and `URI="…"`); verified 1080p in Chromium hls.js. Fall back to the MP4 on any fatal hls.js error.
- Caption tracks: when a Chinese video also has uploaded English captions, "prefer English when the spoken language is unknown" picks English. Infer the spoken language from the title's script (Han → zh, kana → ja, hangul → ko) before falling back. A foreign video's uploaded Chinese track is better than machine translation: align it to the original segments by midpoint and use it as the translation, machine-translating only the gaps. `&tlang=` (YouTube's own auto-translation) returned 429 with a bot page: not usable as a fallback.

## Downloads and the website

- Fixed "always latest" links: `https://github.com/<o>/<r>/releases/latest/download/<App>-mac-apple-silicon.dmg` (upload stable-named copies to every release) and, on your own server, `/download/latest/<stable name>` symlinks to `/download/<version>/…` (`ln -sfn`). Curl every link for 200 after publishing.
- A website deploy with `rsync --delete` will wipe the release files under `/download`. Exclude `/download/latest.json`, `/download/latest/` and the version folders, and prove it with a dry run before the next deploy.
- On aaPanel the site root is `/www/wwwroot/<domain>`, not `/var/www`; read the vhost before writing paths. When the site and an API share a domain (`/api/` proxy), merge into the vhost instead of replacing it and re-check `/api/health`.

## Before calling it shipped

Windows exe import check green and the installer carrying its DLLs, Mac DMGs stapled and `spctl`-checked (also after adding a quarantine attribute), four updater signatures verified, `latest.json` version correct on the server and on GitHub, fixed links 200, an in-app update from the previous version actually done, Windows/Linux listed as "CI-built, not tested on hardware" unless they were.
