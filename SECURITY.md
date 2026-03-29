# Security Policy

> Status: **ACTIVE** — Authoritative security documentation for MacParakeet

## Reporting Vulnerabilities

If you discover a security vulnerability, please report it responsibly:

1. **Do NOT open a public GitHub issue.**
2. Email **security@macparakeet.com** with details.
3. Include steps to reproduce, affected versions, and potential impact.
4. We will acknowledge within 48 hours and provide a fix timeline.

## Security Audit Summary

This document summarizes the security posture of MacParakeet based on a comprehensive review of the codebase (v0.4, ~141 source files, ~70 test files, 963 tests).

### Architecture Overview

MacParakeet is a **local-first** macOS voice application. Speech recognition runs entirely on-device via Apple's Neural Engine (CoreML). Audio data never leaves the device for transcription purposes.

```
┌─────────────────────────────────────────────────┐
│  macOS App (Swift/SwiftUI)                      │
│  ┌──────────┐  ┌──────────┐  ┌───────────────┐ │
│  │ Dictation │  │ File     │  │ YouTube       │ │
│  │ (mic)     │  │ Transcr. │  │ Download      │ │
│  └─────┬─────┘  └────┬─────┘  └───────┬───────┘ │
│        │              │                │         │
│        └──────────────┼────────────────┘         │
│                       ▼                          │
│              ┌────────────────┐                   │
│              │ Parakeet TDT   │ ← Neural Engine  │
│              │ (CoreML/ANE)   │   (on-device)    │
│              └────────────────┘                   │
│                       │                          │
│              ┌────────▼────────┐                  │
│              │ SQLite (GRDB)   │ ← Local storage │
│              └─────────────────┘                  │
└─────────────────────────────────────────────────┘
         │              │              │
    Optional        Optional       Optional
    network:        network:       network:
    LLM APIs     License API    Telemetry
  (user keys)   (LemonSqueezy) (anonymous)
```

### 1. Data Privacy

| Category | Assessment | Details |
|----------|------------|---------|
| Audio processing | ✅ **Local-only** | All STT runs on the Neural Engine via CoreML. Audio never leaves the device. |
| Database | ✅ **Local file** | SQLite via GRDB stored in `~/Library/Application Support/MacParakeet/`. |
| Clipboard | ✅ **Transparent** | Save/restore pattern preserves prior clipboard contents after paste. |
| Temp files | ✅ **Cleaned up** | Audio temp files in `$TMPDIR/macparakeet/` are cleaned after processing. |
| User accounts | ✅ **None** | No login, no email collection, no user tracking. |

### 2. Network Communication

All network communication is **opt-in** or **user-initiated**:

| Connection | Purpose | When | Data Sent |
|------------|---------|------|-----------|
| LLM APIs (OpenAI, Anthropic, etc.) | Transcript summarization & chat | Only when user configures an LLM provider | Transcript text + user's API key (to their chosen provider) |
| LemonSqueezy API | License activation/validation | Only when user enters a license key | License key + instance ID |
| YouTube (yt-dlp) | Audio download | Only when user pastes a YouTube URL | Standard YouTube request |
| Telemetry | Anonymous usage analytics | Opt-out available in Settings | Session-scoped events (no persistent IDs, no content, no IP storage) |
| Feedback | Bug reports | Only when user submits feedback | User-written message + optional email + system info |
| Sparkle | Auto-update check | Periodic | Standard HTTP request to appcast.xml |
| Discover feed | Content recommendations | On app launch (cacheable) | Standard HTTP GET request |

**All network endpoints use HTTPS** (TLS). The only HTTP endpoint is `localhost:11434` for local Ollama inference, which never leaves the machine.

### 3. Credential Storage

| Secret | Storage | Security |
|--------|---------|----------|
| LLM API keys | macOS Keychain (`com.macparakeet.llm`) | `kSecClassGenericPassword` via Security framework |
| License keys | macOS Keychain | Same Keychain-backed store |
| Instance IDs | macOS Keychain | Same Keychain-backed store |

**Key design decisions:**
- API keys are **excluded from Codable serialization** (`CodingKeys` in `LLMProviderConfig` omits `apiKey`), preventing accidental persistence to UserDefaults or logs.
- Per-provider Keychain keys (`llm_api_key_{provider}`) ensure switching providers preserves all saved keys.
- No credentials are hardcoded in source code (verified via automated grep for common key patterns).

### 4. Process Execution

| Binary | Source | Verification |
|--------|--------|-------------|
| yt-dlp | GitHub Releases (`yt-dlp/yt-dlp`) | **SHA-256 checksum verified** against `SHA2-256SUMS` from the same release |
| FFmpeg | Bundled static build | Bundled at build time; SHA-256 verified during build script |
| Node.js | Official Node.js releases | Bundled at build time for yt-dlp JavaScript extraction |

**Process safety:**
- All external processes use Swift's `Process` class with explicit `arguments` arrays (no shell interpolation).
- YouTube URLs are validated against a strict allowlist (`youtube.com`, `www.youtube.com`, `m.youtube.com`, `youtu.be`) with video ID format validation before being passed to yt-dlp.
- yt-dlp arguments use `"--"` separator before the URL to prevent argument injection.
- Binary installation uses atomic file replacement (staged binary → swap) to prevent corruption.

### 5. macOS Permissions

| Permission | Purpose | When Requested |
|------------|---------|----------------|
| Microphone (`com.apple.security.device.audio-input`) | Dictation audio capture | First dictation use |
| Accessibility (`AXIsProcessTrusted`) | Global hotkey + paste simulation (Cmd+V) | First dictation use |
| Network (`com.apple.security.network.client`) | LLM APIs, license validation, YouTube downloads | Always available (opt-in use) |

**The app does NOT request:**
- Full Disk Access
- Screen Recording
- Location
- Contacts/Calendar
- Camera
- System-wide event monitoring beyond Accessibility

### 6. Code Signing & Distribution

- **Hardened Runtime** enabled (prevents code injection, dylib hijacking)
- **Developer ID Application** certificate (Apple-verified identity)
- **Notarized by Apple** (malware scan + stapled ticket)
- **Sparkle 2** auto-updates with **EdDSA signatures** (not just HTTPS)
- **DMG distribution** with drag-to-install (no installer scripts with elevated privileges)

### 7. Telemetry

Telemetry is **anonymous** and **opt-out** (toggle in Settings):

**What IS collected:**
- Feature usage counts (dictation started, export used, etc.)
- Performance metrics (model load time, transcription speed)
- Error types (classified, not raw messages)
- Session duration
- App version, macOS version, chip type, locale

**What is NOT collected:**
- Transcript content or audio
- File names or paths
- API keys or credentials
- IP addresses (not stored server-side per ADR-012)
- Persistent device identifiers (session ID is ephemeral, regenerated each launch)
- User email or personal information

### 8. Dependencies

All dependencies are pinned to specific versions in `Package.resolved`:

| Dependency | Version | Purpose | Vulnerability Check |
|------------|---------|---------|-------------------|
| GRDB.swift | 7.10.0 | SQLite database | ✅ No known vulnerabilities |
| FluidAudio | 0.12.5 | CoreML STT engine | ✅ No known vulnerabilities |
| Sparkle | 2.9.0 | Auto-updates | ✅ No known vulnerabilities |
| swift-argument-parser | 1.7.1 | CLI arguments | ✅ No known vulnerabilities |
| swift-crypto | 4.3.0 | Cryptographic operations | ✅ No known vulnerabilities |
| swift-nio | 2.96.0 | Network I/O | ✅ No known vulnerabilities |
| EventSource | 1.4.1 | SSE streaming | ✅ No known vulnerabilities |
| yyjson | 0.12.0 | JSON parsing | ✅ No known vulnerabilities |

### 9. Test Coverage

963 tests cover security-relevant areas:
- **Database tests**: CRUD operations, migration integrity, SQL injection prevention (via GRDB parameterized queries)
- **Licensing tests**: Activation, validation, deactivation, edge cases
- **LLM client tests**: Request building, API key header injection, error handling
- **Service tests**: All services have protocol-based mocks for isolated testing
- **URL validation tests**: YouTube URL parsing, injection prevention

### 10. Known Limitations

| Area | Status | Mitigation |
|------|--------|------------|
| SQLite not encrypted at rest | By design | macOS FileVault encrypts the full disk; SQLite encryption adds complexity without meaningful security gain for a single-user desktop app |
| Keychain access without `kSecUseDataProtectionKeychain` | Noted in code comments | Requires entitlements not available in SPM dev builds; production builds use hardened runtime |
| Telemetry enabled by default | Opt-out available | Clearly documented in onboarding; single toggle to disable |
| yt-dlp auto-update from GitHub | Weekly, non-blocking | SHA-256 checksum verified against release checksums; failures are silently retried |

## For Security Researchers

If you're auditing this project, the key files to examine are:

| Area | File(s) |
|------|---------|
| Credential storage | `Sources/MacParakeetCore/Licensing/KeychainKeyValueStore.swift` |
| API key management | `Sources/MacParakeetCore/Services/LLMConfigStore.swift` |
| Network requests | `Sources/MacParakeetCore/Services/LLMClient.swift` |
| License API | `Sources/MacParakeetCore/Licensing/LemonSqueezyLicenseAPI.swift` |
| Process execution | `Sources/MacParakeetCore/Services/BinaryBootstrap.swift`, `YouTubeDownloader.swift` |
| Clipboard | `Sources/MacParakeetCore/Services/ClipboardService.swift` |
| Telemetry | `Sources/MacParakeetCore/Services/TelemetryService.swift` |
| URL validation | `Sources/MacParakeetCore/Utilities/YouTubeURLValidator.swift` |
| Permissions | `Sources/MacParakeetCore/Services/PermissionService.swift` |
| Database | `Sources/MacParakeetCore/Database/DatabaseManager.swift` |
| Entitlements | `scripts/dist/MacParakeet.entitlements` |
| Code signing | `scripts/dist/sign_notarize.sh` |

## License

MacParakeet is free and open-source software licensed under [GPL-3.0](LICENSE). The source code is publicly auditable.
