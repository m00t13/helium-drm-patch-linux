# helium-drm-patch

A lightweight, zero-dependency Python CLI to enable **Widevine CDM (DRM)** playback in [Helium Browser](https://helium.computer/) on Linux—without needing Google Chrome installed, without `bun`, and without `sudo`.

## Why This Exists

Helium is an ungoogled-Chromium-based browser and does not ship with Google's proprietary Widevine Content Decryption Module (`WidevineCdm`). Existing cross-platform tools often require `bun`, expect a local Google Chrome install on Linux, or copy files into root-owned system directories like `/opt/helium-browser-bin/` where package updates can overwrite them.

`helium-drm-patch` installs Widevine purely in your user profile directory (`~/.config/net.imput.helium/WidevineCdm`) using Chromium's native component-update hint file, so it survives browser upgrades cleanly.

## Requirements

- **OS:** Linux (`x86_64` / `amd64`, and `aarch64` / `arm64` via Omaha CRX3)
- **Python:** 3.10+ (standard library only; no `pip` packages required)
- **Optional:** `bsdtar` (only used on `x86_64` if the Google Chrome `.deb` fallback is needed)

## Quick Start

```bash
git clone https://github.com/m00t13/helium-drm-patch.git
cd helium-drm-patch
chmod +x helium-drm-patch
./helium-drm-patch --verify
```

Or install it to `~/.local/bin` so you can run `helium-drm-patch` anytime to update Widevine:

```bash
install -Dm755 helium-drm-patch ~/.local/bin/helium-drm-patch
helium-drm-patch --verify
```

After running, **restart Helium** if it was open. You can confirm Widevine is registered at `helium://components` or test playback at <https://bitmovin.com/demos/drm>.

## CLI Options

| Flag | Description |
| --- | --- |
| `--verify` | Run `ldd` and an isolated headless Helium EME probe (`com.widevine.alpha` for VP9, H.264, and AAC) after installing. |
| `--verify-only` | Verify the currently installed Widevine CDM without downloading or modifying files. |
| `--force` | Re-download and reinstall even if the latest version is already installed. |
| `--chrome-widevine <dir>` | Copy Widevine from an existing local `WidevineCdm` directory instead of downloading. |
| `--user-data-dir <dir>` | Override Helium's user data directory (default: `${XDG_CONFIG_HOME:-~/.config}/net.imput.helium`). |
| `--helium-binary <path>` | Specify the Helium executable path for `--verify` / `--verify-only` if auto-detection doesn't find it. |

## How It Works

1. Reads Helium's `Local State` JSON to detect the underlying Chromium version (falling back to a recent default if Helium hasn't been launched yet).
2. Queries Google's Omaha component update API (`update.googleapis.com/service/update2/json`) for the official Widevine CDM CRX3 package (`oimompecagnajdejgnnjijobebaeigek`), verifies its SHA-256 checksum, parses the CRX3 header, and extracts the payload safely. On `x86_64`, falls back to extracting `WidevineCdm` from Google's official Chrome `.deb` if Omaha fails.
3. Validates `manifest.json` and the ELF binary (`_platform_specific/linux_<arch>/libwidevinecdm.so`) and installs them atomically into `<user-data-dir>/WidevineCdm/<version>/`, pruning stale older versions.
4. Writes `<user-data-dir>/WidevineCdm/latest-component-updated-widevine-cdm` pointing to the installed version directory—the native hint file Chromium reads at startup.

## Compatibility & Limitations

- **Tested environment:** Arch Linux (`helium-browser-bin`, Chromium 158, `x86_64`). Should work on any Linux distribution using the standard XDG config directory (or pass `--user-data-dir` for Flatpak / custom profile paths).
- **Widevine L3 vs. VMP / Hardware L1:** On desktop Linux, this provides **Widevine L3** software decryption. Services like Spotify, YouTube, Crunchyroll, Bitmovin, and many streaming endpoints work out of the box. However, providers that strictly enforce cryptographic **Verified Media Path (VMP)** attestation against a commercially signed browser binary—or hardware **Widevine L1 / HDCP** (e.g., 4K/HDR on Netflix, Amazon Prime Video, or Disney+)—may still restrict playback or cap resolution.

## Updating & Uninstalling

- **Update Widevine:** Re-run `helium-drm-patch --verify` whenever you want to pull the latest Widevine CDM release.
- **Uninstall:** Remove the user-space Widevine directory:
  ```bash
  rm -rf ~/.config/net.imput.helium/WidevineCdm
  ```

## Acknowledgments & Disclaimer

- Inspired by [vikas5914/helium-drm-fixer](https://github.com/vikas5914/helium-drm-fixer).
- This repository does not host or redistribute any proprietary Google/Widevine binaries; `helium-drm-patch` fetches the official module directly from Google's servers at runtime.
