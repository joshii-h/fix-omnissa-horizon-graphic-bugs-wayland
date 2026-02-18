# Fix Omnissa Horizon Client Graphical Bugs on Wayland

Omnissa Horizon Client on Linux has multiple graphical bugs when running under **Wayland** (via XWayland). This repository documents all known issues and provides ready-to-use patches for **Fedora**, **Gentoo**, and **Arch Linux**.

---

## Recommended: Use Horizon Client Next (2512+)

Starting with version **2512**, Omnissa ships a second client called **Horizon Client Next** alongside the classic client. It is a complete rewrite using **.NET with Avalonia UI** (Skia/Vulkan rendering) instead of GTK3.

**Horizon Client Next does not suffer from the GTK3/XWayland bugs** documented below. It handles Wayland, HiDPI scaling, and dark themes natively. If you are on version 2512 or later, **use Horizon Client Next instead of applying the patches in this repo**.

### Making Horizon Client Next the Default

The 2512 installer registers both clients but keeps the classic client as the default. To switch:

```bash
# Create local desktop entry overrides
mkdir -p ~/.local/share/applications

# Hide the classic client from the app launcher
cat > ~/.local/share/applications/horizon-client.desktop << 'EOF'
[Desktop Entry]
Encoding=UTF-8
Type=Application
NoDisplay=true
Name=Omnissa Horizon Client (Classic)
Exec=horizon-client %u
Icon=/usr/share/icons/horizon-client.png
EOF

# Make Next the default with proper URI handlers
cat > ~/.local/share/applications/horizon-client-next.desktop << 'EOF'
[Desktop Entry]
Encoding=UTF-8
Type=Application
Icon=/usr/share/icons/horizon-client.png
Exec=horizon-client-next %u
MimeType=x-scheme-handler/horizon-client;x-scheme-handler/vmware-view;
Categories=Application;Network;
Actions=about-window;
Name=Omnissa Horizon Client

[Desktop Action about-window]
Name=About
Exec=horizon-client-next --about --useExisting
EOF

# Update desktop database and register URI handlers
update-desktop-database ~/.local/share/applications/
xdg-mime default horizon-client-next.desktop x-scheme-handler/horizon-client x-scheme-handler/vmware-view
```

After this, "Omnissa Horizon Client" in your app launcher will start the Next client, and `vmware-view://` / `horizon-client://` URIs will be handled by it.

> **Note:** The classic client remains available at `/usr/bin/horizon-client` if needed.

---

## Classic Client Fixes (2412 and earlier, or 2512 without Next)

The sections below apply only to the **classic GTK3-based client**. If you are using Horizon Client Next, you can skip everything below.

## Quick Reference

| Fix | Issue | Applies to you if... | Patch label |
|---|---|---|---|
| [Bug 1](#bug-1-client-crashes-after-connecting-libx11-xkb-bug) | XKB crash | libxkbcommon >= 1.12 **and** libX11 < 1.8.13 | separate libX11 patch |
| [Fix 2a](#fix-2a-dark-theme) | White-on-white text | You use a **dark GTK theme** (Breeze Dark, Adwaita Dark, etc.) | `GTK_THEME=Adwaita` |
| [Fix 2b](#fix-2b-hidpi-scaling) | UI too small | You have a **HiDPI display** (4K, Retina, scaling > 1x) | `GDK_SCALE=2` |
| [Fix 2c](#fix-2c-wayland-warning-dialog) | "Protocol not supported" dialog | You run a **Wayland session** (KDE Plasma Wayland, GNOME Wayland, etc.) | `XDG_SESSION_TYPE` + bwrap |
| [Bug 3](#bug-3-missing-libraries-gtkmm-libclientsdkcprimitive) | Missing PCoIP libs | PCoIP tarball **not installed** alongside the main bundle | manual install step |

---

## Bug 1: Client Crashes After Connecting (libX11 XKB Bug)

**Severity:** Critical -- client is completely unusable without this fix.

**Applies to you if:** Your system has **libxkbcommon >= 1.12** and **libX11 < 1.8.13**.

**Does NOT apply if:** libxkbcommon < 1.12 (bug not triggered) or libX11 >= 1.8.13 (fix already included upstream).

> **Note:** This bug affects both the classic client and Horizon Client Next, since both use `horizon-protocol` which links against libX11.

```bash
# Check your versions
rpm -q libX11 libxkbcommon        # Fedora
qlist -Iv x11-libs/libX11 x11-libs/libxkbcommon  # Gentoo
pacman -Q libx11 libxkbcommon     # Arch
```

### Symptoms

- Client crashes/closes immediately when the VDI session window receives mouse focus
- Crashes when mousing over the toolbar
- Segfault at `horizon-protocol`
- X errors: `BadValue ... related to the XKEYBOARD extension`

### Root Cause

When libxkbcommon >= 1.12 sends an `XkbNewKeyboardNotify` event for a device that libX11 hasn't loaded a keymap for yet, libX11 dereferences a NULL pointer. Fixed upstream in [libX11 1.8.13](https://gitlab.freedesktop.org/xorg/lib/libx11/-/merge_requests/293).

### Affected Distributions

| Distribution | libxkbcommon | libX11 | Status |
|---|---|---|---|
| Gentoo (rolling) | >= 1.12 | < 1.8.13 | **Affected** -- apply patch below |
| Arch Linux (rolling) | >= 1.12 | >= 1.8.13 | **Fixed** in repos since Feb 2026 |
| Fedora 43 | 1.11.0 | 1.8.12 | **Not yet affected** -- monitor after updates |

### Fix: Gentoo

```bash
sudo mkdir -p /etc/portage/patches/x11-libs/libX11
sudo cp patches/libx11-mr293-fix-xkb-crash.patch /etc/portage/patches/x11-libs/libX11/
sudo emerge -1 x11-libs/libX11
```

### Fix: Arch Linux

libX11 >= 1.8.13 is in the repos since February 2026. Update with `pacman -Syu`.

If still on an older version: `yay -S libx11-mr293` ([AUR](https://aur.archlinux.org/packages/libx11-mr293))

### Workaround

Switching to a US keyboard layout can sometimes mitigate the crash. Not a reliable fix.

---

## Bug 2: Launcher Script Fixes (Dark Theme / HiDPI / Wayland Warning)

> **Skip this section if you are using Horizon Client Next** -- these are GTK3-specific issues.

The launcher script `/usr/bin/horizon-client` needs several environment variable additions depending on your setup. The patches in this repo bundle all three fixes with clear labels. **After patching, edit the file and remove any fixes that do not apply to you.**

> **Note:** These patches are overwritten on every Horizon Client update. Re-apply after updating.

### Fix 2a: Dark Theme

**Applies to you if:** You use a **dark GTK theme** (KDE Breeze Dark, GNOME Adwaita Dark, any theme with `gtk-application-prefer-dark-theme=true`).

**Does NOT apply if:** You use a light GTK theme.

**What it does:** Sets `GTK_THEME=Adwaita` to force the light Adwaita theme for the Horizon Client only. Your desktop stays dark.

**Why:** Horizon Client uses hardcoded Pango markup colors designed for light backgrounds. With a dark theme, all text becomes white-on-white (invisible).

### Fix 2b: HiDPI Scaling

**Applies to you if:** You have a **HiDPI/4K display** and the client UI appears tiny/unreadable.

**Does NOT apply if:** You use standard-DPI displays (1080p, 1440p at 100% scaling) where the client looks fine.

**What it does:** Sets `GDK_SCALE=2` to double all UI elements.

**Why:** GTK3 under XWayland does not support fractional scaling. The only options are 1x (too small on HiDPI) or 2x (slightly large but readable). There is no in-between.

**Note:** This fix is **commented out by default** in the patches. Uncomment `export GDK_SCALE=2` in `/usr/bin/horizon-client` if you need it.

### Fix 2c: Wayland Warning Dialog

**Applies to you if:** You run a **Wayland session** and get the dialog: *"The display server protocol that you are using is not supported."*

**Does NOT apply if:** You run an **X11 session** (`echo $XDG_SESSION_TYPE` shows `x11`).

**What it does:** Two-stage suppression:
1. Sets `XDG_SESSION_TYPE=x11` and unsets `WAYLAND_DISPLAY` (env var level)
2. Uses [bubblewrap](https://github.com/containers/bubblewrap) (`bwrap`) to present a modified systemd login session file to the client binary, because **the client reads `/run/systemd/sessions/<id>` directly** and ignores environment variables

**Requires:** `bubblewrap` package (already installed if you use Flatpak). Falls back gracefully if not available -- the warning dialog appears but the client still works.

### Applying the Patches

```bash
# Fedora (Horizon Client 2512 RPM launcher format)
sudo patch -p1 -d / < patches/horizon-client-launcher-fedora.patch

# Gentoo / Arch Linux (Horizon Client 2512 bundle launcher format)
sudo patch -p1 -d / < patches/horizon-client-launcher.patch
```

After patching, review `/usr/bin/horizon-client` and remove any fixes that do not apply to your setup (each fix is marked with `[Fix 2a]`, `[Fix 2b]`, or `[Fix 2c]`).

---

## Bug 3: Missing Libraries (gtkmm, libclientSdkCPrimitive)

**Applies to you if:** The client fails to start with errors about missing `libgtkmm-3.0.so.1` or `libclientSdkCPrimitive.so`.

**Does NOT apply if:** The client starts without library errors.

### Fix

The Horizon Client consists of two separate downloads. If only the main bundle is installed, PCoIP libraries are missing.

1. Download **both** the main client bundle and the PCoIP tarball from [Omnissa](https://customerconnect.omnissa.com/downloads/info/slug/desktop_end_user_computing/omnissa_horizon_client_for_linux/2512).

2. Install the main bundle:

```bash
chmod +x Omnissa-Horizon-Client-*.bundle
sudo env TERM=dumb VMWARE_EULAS_AGREED=yes ./Omnissa-Horizon-Client-*.bundle --console --required
```

3. Extract PCoIP libraries:

```bash
sudo tar xzf Omnissa-Horizon-PCoIP-*.tar.gz -C /usr/lib/omnissa/horizon/
```

4. If needed, create symlink:

```bash
sudo ln -sf /usr/lib/omnissa/horizon/lib/libclientSdkCPrimitive.so /usr/lib/libclientSdkCPrimitive.so
```

---

## Full Installation Guide

### Fedora

```bash
# 1. Install the Horizon Client RPM (includes both classic and Next client)
sudo dnf install ./Omnissa-Horizon-Client-2512-*.x64.rpm

# 2. (Recommended) Make Horizon Client Next the default -- see instructions above

# 3. (Classic client only) Apply launcher fixes
sudo patch -p1 -d / < patches/horizon-client-launcher-fedora.patch

# 4. (Optional) Disable telemetry
mkdir -p ~/.omnissa
cp configs/horizon-preferences.example ~/.omnissa/horizon-preferences
```

Bug 1 (XKB crash) is not yet triggered on Fedora 43 (libxkbcommon 1.11.0 < 1.12). Monitor after system updates.

### Gentoo

```bash
# 1. Install the Horizon Client bundle
chmod +x Omnissa-Horizon-Client-*.bundle
sudo env TERM=dumb VMWARE_EULAS_AGREED=yes ./Omnissa-Horizon-Client-*.bundle --console --required

# 2. (Recommended) Make Horizon Client Next the default -- see instructions above

# 3. Apply libX11 XKB crash fix (Bug 1) if affected
sudo mkdir -p /etc/portage/patches/x11-libs/libX11
sudo cp patches/libx11-mr293-fix-xkb-crash.patch /etc/portage/patches/x11-libs/libX11/
sudo emerge -1 x11-libs/libX11

# 4. (Classic client only) Apply launcher fixes
sudo patch -p1 -d / < patches/horizon-client-launcher.patch

# 5. (Optional) Disable telemetry
mkdir -p ~/.omnissa
cp configs/horizon-preferences.example ~/.omnissa/horizon-preferences
```

---

## Disable Telemetry

Set the following in `~/.omnissa/horizon-preferences`:

```
view.enableDataSharing = 'FALSE'
```

See `configs/horizon-preferences.example` for a full recommended configuration.

---

## Known Limitations

### Classic Client (GTK3)

- **No native Wayland support** -- Horizon Client runs under XWayland (`GDK_BACKEND=x11`). Omnissa has not implemented native Wayland support.

- **No fractional HiDPI scaling** -- GTK3 only supports integer scaling (`GDK_SCALE=1` or `GDK_SCALE=2`). There is no way to set e.g. 1.5x scaling.

- **Multi-monitor flicker** -- Some users report flickering when using multiple monitors under XWayland. This is an XWayland limitation.

- **Launcher patches not persistent** -- The fixes in `/usr/bin/horizon-client` are overwritten on every Horizon Client update. Re-apply the patch after updating.

### Horizon Client Next (Avalonia)

- **New in 2512** -- Horizon Client Next is relatively new. Report issues to Omnissa if you encounter problems.

- **URI handlers not registered by default** -- The installer does not register `vmware-view://` and `horizon-client://` URI schemes for the Next client. Use the desktop entry override above to fix this.

---

## References

- [libX11 MR #293: Fix XKB crash](https://gitlab.freedesktop.org/xorg/lib/libx11/-/merge_requests/293)
- [libxkbcommon issue #888](https://github.com/xkbcommon/libxkbcommon/issues/888)
- [Debian Bug #1120988](http://www.mail-archive.com/debian-x@lists.debian.org/msg146932.html)
- [AUR: libx11-mr293](https://aur.archlinux.org/packages/libx11-mr293)
- [AUR: omnissa-horizon-client](https://aur.archlinux.org/packages/omnissa-horizon-client)

---

## License

[GPL-3.0-or-later](LICENSE)
