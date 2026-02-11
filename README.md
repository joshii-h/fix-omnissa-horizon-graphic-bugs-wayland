# Fix Omnissa Horizon Client Graphical Bugs on Wayland

Omnissa Horizon Client **2512** (8.17.0) on Linux has multiple graphical bugs when running under **Wayland** (via XWayland), particularly on rolling-release distributions like **Gentoo** and **Arch Linux**. This repository documents all known issues and provides ready-to-use patches.

## Table of Contents

- [Bug 1: Client Crashes After Connecting (libX11 XKB Bug)](#bug-1-client-crashes-after-connecting-libx11-xkb-bug)
- [Bug 2: White Text on White Background (Dark Theme)](#bug-2-white-text-on-white-background-dark-theme)
- [Bug 3: Missing Libraries (gtkmm, libclientSdkCPrimitive)](#bug-3-missing-libraries-gtkmm-libclientsdkcprimitive)
- [Installation Guide (Gentoo)](#installation-guide-gentoo)
- [Disable Telemetry](#disable-telemetry)
- [Known Limitations](#known-limitations)
- [References](#references)

---

## Bug 1: Client Crashes After Connecting (libX11 XKB Bug)

**Severity:** Critical -- client is completely unusable without this fix.

### Symptoms

- Client crashes/closes immediately when the VDI session window receives mouse focus
- Crashes when mousing over the toolbar
- Segfault at the protocol binary (`horizon-protocol`)
- X Window System errors in terminal output:

```
The program 'horizon-client' received an X Window System error.
This probably reflects a bug in the program.
The error was 'BadValue (integer parameter out of range for operation)'
...related to the XKEYBOARD extension.
```

- GTK assertion warnings before the crash:

```
gtk_window_present_with_time: assertion 'GTK_IS_WINDOW (window)' failed
g_source_remove: assertion 'tag > 0' failed
g_object_ref: assertion 'G_IS_OBJECT (object)' failed
```

### Root Cause

When **libxkbcommon >= 1.12** sends an `XkbNewKeyboardNotify` event for a device that libX11 hasn't loaded a keymap for yet, libX11 dereferences a NULL pointer in `XKBBind.c` and `XKBUse.c`. This is a race condition between libxkbcommon and libX11 that affects any XKB-aware application running under XWayland.

The fix is upstream in [libX11 merge request #293](https://gitlab.freedesktop.org/xorg/lib/libx11/-/merge_requests/293).

### Fix: Gentoo (User Patches)

Copy the patch into Gentoo's user patches directory. It will be applied automatically the next time `x11-libs/libX11` is rebuilt:

```bash
sudo mkdir -p /etc/portage/patches/x11-libs/libX11
sudo cp patches/libx11-mr293-fix-xkb-crash.patch /etc/portage/patches/x11-libs/libX11/
sudo emerge -1 x11-libs/libX11
```

### Fix: Arch Linux (AUR)

Install the patched libX11 from the AUR:

```bash
yay -S libx11-mr293
```

See: [AUR: libx11-mr293](https://aur.archlinux.org/packages/libx11-mr293)

### Workaround

Switching to a US keyboard layout can sometimes mitigate the crash if you are using a non-US layout, since the bug is triggered by XKB keyboard mapping changes. This is not a reliable fix.

---

## Bug 2: White Text on White Background (Dark Theme)

### Symptoms

- All text in the Horizon Client UI is invisible (white on white)
- Buttons and menu items appear blank
- Occurs when using a dark GTK theme (e.g., KDE Plasma with Breeze Dark)

### Root Cause

The Horizon Client uses **hardcoded Pango markup colors** designed for light backgrounds. When a dark GTK theme is active (e.g., `gtk-application-prefer-dark-theme=true` in `~/.config/gtk-3.0/settings.ini`), the background becomes dark but the hardcoded text colors remain light/white, making text invisible.

### Fix

Add `export GTK_THEME=Adwaita` to the launcher script `/usr/bin/horizon-client`, forcing the light Adwaita theme for the client only (your desktop stays dark):

**Manual fix:**

```bash
sudo sed -i '/export GDK_BACKEND=x11/a\\n# Force light GTK theme - Horizon'\''s hardcoded Pango markup colors assume\n# a light background. Dark themes cause white-on-white text.\nexport GTK_THEME=Adwaita' /usr/bin/horizon-client
```

**Or apply the patch:**

```bash
sudo patch -p1 -d / < patches/horizon-client-launcher.patch
```

> **Note:** This patch will need to be re-applied after every Horizon Client update, since the installer overwrites `/usr/bin/horizon-client`.

---

## Bug 3: Missing Libraries (gtkmm, libclientSdkCPrimitive)

### Symptoms

- Client fails to start with errors about missing `libgtkmm-3.0.so.1`, `libatkmm-1.6.so.1`, or similar gtkmm libraries
- Errors referencing `libclientSdkCPrimitive.so` (error 4 -- unmapped memory)
- PCoIP protocol fails to initialize

### Root Cause

The Horizon Client consists of **two separate tarballs**:

1. **Main client bundle** (`Omnissa-Horizon-Client-*.bundle`) -- the client UI
2. **PCoIP tarball** -- contains additional protocol libraries including gtkmm bindings and `libclientSdkCPrimitive.so`

If only the main bundle is installed, the PCoIP libraries are missing.

### Fix

1. Download **both** the main client bundle and the PCoIP tarball from the [Omnissa download page](https://customerconnect.omnissa.com/downloads/info/slug/desktop_end_user_computing/omnissa_horizon_client_for_linux/2512).

2. Install the main bundle first:

```bash
chmod +x Omnissa-Horizon-Client-*.bundle
sudo env TERM=dumb VMWARE_EULAS_AGREED=yes ./Omnissa-Horizon-Client-*.bundle --console --required
```

3. Extract the PCoIP tarball into the client library directory.

4. If `libclientSdkCPrimitive.so` is not found at runtime, create a symlink:

```bash
sudo ln -sf /usr/lib/omnissa/horizon/lib/libclientSdkCPrimitive.so /usr/lib/libclientSdkCPrimitive.so
```

---

## Installation Guide (Gentoo)

There is no official Portage ebuild for Omnissa Horizon Client. Install from the tarball:

```bash
# 1. Download the bundle from Omnissa
# 2. Make it executable
chmod +x Omnissa-Horizon-Client-*.bundle

# 3. Install (non-interactive)
sudo env TERM=dumb VMWARE_EULAS_AGREED=yes ./Omnissa-Horizon-Client-*.bundle --console --required

# 4. Apply the libX11 XKB crash fix
sudo mkdir -p /etc/portage/patches/x11-libs/libX11
sudo cp patches/libx11-mr293-fix-xkb-crash.patch /etc/portage/patches/x11-libs/libX11/
sudo emerge -1 x11-libs/libX11

# 5. Apply the GTK theme fix
sudo patch -p1 -d / < patches/horizon-client-launcher.patch

# 6. (Optional) Disable telemetry
mkdir -p ~/.omnissa
cp configs/horizon-preferences.example ~/.omnissa/horizon-preferences
# Edit to set your broker URL and preferences
```

---

## Disable Telemetry

Omnissa Horizon Client has a **Customer Experience Improvement Program (CEIP)** that sends usage data. To disable it, set the following in `~/.omnissa/horizon-preferences`:

```
view.enableDataSharing = 'FALSE'
```

See `configs/horizon-preferences.example` for a full recommended configuration.

---

## Known Limitations

- **No native Wayland support** -- Horizon Client runs under XWayland (`GDK_BACKEND=x11`). Omnissa has not implemented native Wayland support yet. You will see this warning on startup:

  > "The display server protocol you are using is not supported. It is recommended to connect with the x11 display server protocol."

- **Multi-monitor flicker** -- Some users report flickering when using multiple monitors under XWayland. This is an XWayland limitation, not specific to Horizon Client.

- **Launcher patch not persistent** -- The GTK_THEME fix in `/usr/bin/horizon-client` is overwritten on every Horizon Client update. Re-apply the patch after updating.

---

## References

- [libX11 MR #293: Fix XKB crash](https://gitlab.freedesktop.org/xorg/lib/libx11/-/merge_requests/293)
- [libxkbcommon issue #888](https://github.com/xkbcommon/libxkbcommon/issues/888)
- [Debian Bug #1120988](http://www.mail-archive.com/debian-x@lists.debian.org/msg146932.html)
- [AUR: libx11-mr293](https://aur.archlinux.org/packages/libx11-mr293)
- [AUR: omnissa-horizon-client](https://aur.archlinux.org/packages/omnissa-horizon-client)

## License

[MIT](LICENSE)
