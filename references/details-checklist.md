# Details Checklist (what users don't think to ask for)

Walk this before every handoff. Items marked ★ are in the template already; verify they still work.

## First run

- [ ] ★ Something visible happens on first launch (settings with checklist, or the main window empty state).
- [ ] Permissions requested only when needed, explained first, with a working "denied" path.
- [ ] Zero-config default destination/behaviour works for someone with no special setup.
- [ ] If the app runs from the DMG or Downloads (translocated), tell the user to move it to Applications (Sparkle and login items need it).
- [ ] ★ Launch at login default matches the shape (menu bar on, window off) and is re-applied after moving the app.

## Everyday use

- [ ] Every action has keyboard access; shortcuts are shown in menus and tooltips.
- [ ] ★ Esc closes, ⌘W closes, ⌘↩ confirms, ⌘, opens settings, from every window.
- [ ] Undo for destructive or saving actions ("已保存 · 撤销").
- [ ] Input is never lost: failed saves keep the text and offer retry; a draft survives closing the panel.
- [ ] Works on multiple displays and Spaces, in full-screen apps, and with the menu bar auto-hidden.
- [ ] Dark mode, increased contrast, Reduce Motion and large text checked.
- [ ] Long strings (Chinese and English) truncate gracefully; numbers and dates use the user's locale except where verbatim matters.
- [ ] Chinese IME: composing text, placeholder, search with pinyin.
- [ ] Drag & drop accepts files, images, URLs, text, and file promises from Photos/Mail.
- [ ] Clipboard: respects transient/concealed markers; images from sandboxed apps (WeChat) work.
- [ ] Sounds are optional and pleasant; respect "play feedback" preferences.

## Settings and About

- [ ] ★ Native toolbar tabs, title follows the tab, nothing auto-focused.
- [ ] Every setting has a default, applies immediately, and shows a preview when it changes output.
- [ ] ★ About: version, 检查更新, 反馈问题, 使用说明, 源代码, auto-update toggle, QR cards, privacy line, license, correct year.
- [ ] A way to reset settings and to reveal data files in Finder.

## Window apps with a content column (learned on SubCast)

- [ ] Destructive actions confirm in an in-app dialog (WebView `confirm()` silently returns false).
- [ ] Resizable panes have min/max widths and remember them; the middle column keeps a minimum so controls don't wrap; narrow panes collapse to icons, never vertical text.
- [ ] Playback/primary controls dock to the bottom when scrolled out of view; mini player aligns with the content column.
- [ ] Popovers near the bottom (copy menus in chat) open upward; dialogs fit without scrollbars.
- [ ] Sticky headers are opaque; scrollbars appear only while scrolling.
- [ ] Menus that only make sense with data (已下载) appear only when there is data.
- [ ] Original names stay original (an English podcast keeps its English title); no Chinese product name in the chrome when a logo + English name is the brand.
- [ ] Theme switches change layout, not only colours; dashes in display text are optically centred or avoided.
- [ ] Works with zero setup: AI features without a key say so in their own empty state; a free fallback (e.g. Google Translate) exists.
- [ ] Settings for ordinary users: credential with a "get a key" link → what it is used for (multi-select chips) → test → advanced (collapsed).
- [ ] Cache/storage page: sizes, open folder, clear.

## Robustness

- [ ] Logic covered by `--selftest`; it doesn't touch real user data.
- [ ] Atomic writes; handles missing/renamed/iCloud-evicted folders with a clear message.
- [ ] No main-thread disk or network work in typing paths.
- [ ] Memory/CPU idle near zero for menu bar apps (no busy polling faster than needed; pause watchers when not needed).
- [ ] Sleep/wake and network changes don't break hotkeys or watchers.
- [ ] Intel + Apple silicon (universal build); minimum macOS actually tested or stated.

## Shipping

- [ ] ★ Signed with Developer ID, hardened runtime, notarized, stapled; Gatekeeper output checked.
- [ ] DMG looked at: background, arrow, names.
- [ ] Appcast verified, an update from the previous version actually tested.
- [ ] Version bumped, release notes written for users, README screenshots current.
- [ ] Repo private until the user says public; then About, topics, social preview set.
- [ ] THIRD_PARTY_NOTICES complete; licenses for bundled fonts/icons/sounds included.
- [ ] `./install-local.sh` run so the user is testing the newest build.
- [ ] Handoff lists what was verified and what was not.
- [ ] Every new update/onboarding component is actually mounted (grep for its import in the app root).
- [ ] Public copy (website, README) checked against the code: model sizes, accelerators, platforms.
- [ ] Delegated website/deploy work: if the goal is "live", the brief authorizes deploy or the main session finishes it.
- [ ] Don't screenshot the user's whole screen to check the app; capture the app window by id, or ask the user to confirm.
