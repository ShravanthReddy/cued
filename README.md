# cued

A system-wide autocomplete utility for macOS. cued shows a small floating popup at your cursor that suggests text completions in most native apps. Tab to accept; Esc to dismiss.

## Status

v0.1 — scaffolded, builds, basic flow wired:

- Single-process menu-bar app (`LSUIElement`).
- Accessibility-driven focus tracking and pause-detected text capture.
- Completion engine with cancellation on focus change or resumed typing.
- Three-provider abstraction (`CompletionProvider`) with implementations:
  - **Apple Foundation Models** (macOS 26+, on-device).
  - **Cloud** (Anthropic Messages API; OpenAI Chat Completions).
  - **MLX** — scaffolded but not yet wired to the runtime. The right packages are `mlx-swift` and `mlx-swift-examples` (which exports `MLXLLM`/`MLXLMCommon`). See `Sources/CuedProviders/MLXProvider.swift` for the wiring steps.
- Floating non-activating popup at the caret with verified AX insertion (pasteboard fallback).
- Settings UI (provider, cloud vendor + model + Keychain-backed key, debounce, denylist).
- Onboarding window deep-linking to System Settings → Accessibility, polling for grant.

## Requirements

- macOS 14+ (deployment target). Foundation Models provider requires macOS 26 with Apple Intelligence enabled.
- Swift 6.0 toolchain. With Command Line Tools alone you can `swift build` but tests and Xcode-based signing require a full Xcode install.

## Build & run

```sh
# Run from source (no .app bundle)
swift run

# Run unit tests (Swift Testing)
swift test

# Assemble an .app bundle for distribution
./Scripts/build-app.sh                          # ad-hoc signed
DEVELOPER_ID="Developer ID Application: …" \
    ./Scripts/build-app.sh                      # production sign

# Notarize after building with DEVELOPER_ID set:
xcrun notarytool submit build/cued.app \
    --apple-id <id> --team-id <TEAMID> --password <app-pw> --wait
xcrun stapler staple build/cued.app
```

## Permissions

cued asks for **Accessibility** access (mandatory). It does **not** request Input Monitoring; it never reads your raw keystrokes. AX notifications are sufficient for both observing edits and writing back the accepted suggestion.

## Architecture

- `Sources/CuedCore` — shared types, `CompletionProvider` protocol, prompt sanitiser.
- `Sources/CuedAccessibility` — `AXPermission`, `FocusTracker`, `ContextExtractor`, `CaretLocator`, `Inserter` (with verified AX insertion + pasteboard fallback), `RoleFilter`, `Denylist`.
- `Sources/CuedEngine` — `Debouncer`, `CompletionEngine` (actor, cancels stale work).
- `Sources/CuedProviders` — `FoundationModelsProvider`, `CloudProvider`, `MLXProvider` (stub), `ProviderFactory`.
- `Sources/CuedUI` — `SuggestionWindow` (NSPanel non-activating), `SuggestionView`, `AcceptKeyHandler` (Carbon `RegisterEventHotKey`, scoped to popup visibility), `SettingsScene`, `OnboardingWindow`.
- `Sources/CuedStorage` — `Settings` (`UserDefaults`), `Keychain` (Security framework), `ModelStore`.
- `Sources/CuedApp` — `AppDelegate`, `MenuBarController`, `main.swift`.

The full design lives in `~/.claude/plans/i-want-to-vast-globe.md`.

## Known limitations

- Electron/web/custom controls (Slack, VS Code, Notion, browser textareas) generally won't return caret bounds via Accessibility — the popup will silently no-op there.
- Tab-as-accept uses a globally registered hotkey while the popup is visible; users can pick a less-conflicting key in Settings.
- Foundation Models is discouraged by Apple for code generation; prefer Cloud (Haiku) or MLX once enabled.
- App Sandbox is incompatible with system-wide Accessibility access; cued ships Developer-ID-direct, not via the App Store.

## License

TBD.
