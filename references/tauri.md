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

## Downloads and the website

- Fixed "always latest" links: `https://github.com/<o>/<r>/releases/latest/download/<App>-mac-apple-silicon.dmg` (upload stable-named copies to every release) and, on your own server, `/download/latest/<stable name>` symlinks to `/download/<version>/…` (`ln -sfn`). Curl every link for 200 after publishing.
- A website deploy with `rsync --delete` will wipe the release files under `/download`. Exclude `/download/latest.json`, `/download/latest/` and the version folders, and prove it with a dry run before the next deploy.
- On aaPanel the site root is `/www/wwwroot/<domain>`, not `/var/www`; read the vhost before writing paths. When the site and an API share a domain (`/api/` proxy), merge into the vhost instead of replacing it and re-check `/api/health`.

## Before calling it shipped

Mac DMGs stapled and `spctl`-checked (also after adding a quarantine attribute), four updater signatures verified, `latest.json` version correct on the server and on GitHub, fixed links 200, an in-app update from the previous version actually done, Windows/Linux listed as "CI-built, not tested on hardware" unless they were.
