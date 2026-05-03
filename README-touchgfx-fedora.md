# TouchGFX Designer 4.26.1 on Fedora Linux (via Wine)

Guide for installing and running TouchGFX Designer under Wine on Fedora 43, including headless CLI code generation for CI/CD workflows.

> **Note:** TouchGFX Designer has no native Linux support. ST employees suggest Wine as the supported workaround. A native Linux CLI is planned but has no release timeline (as of 2026).

---

## Prerequisites

- Fedora 43 (or similar)
- `wine-staging` and `winetricks`
- The TouchGFX Designer MSI installer (`TouchGFX-4.26.1.msi`)

### Install Wine and Winetricks

```bash
sudo dnf install wine-staging winetricks
```

---

## 1. Create a dedicated Wine prefix

Using a dedicated prefix keeps TouchGFX isolated from other Wine apps.

```bash
export WINEPREFIX=~/.wine-touchgfx
WINEARCH=win64 wineboot --init
```

---

## 2. Install .NET Framework 4.8

TouchGFX Designer requires .NET 4.8.

```bash
WINEPREFIX=~/.wine-touchgfx winetricks -q dotnet48
```

This takes several minutes. It will install `mono` stubs and the full .NET runtime.

---

## 3. Install Visual C++ Runtime 2022

Required to fix MSI custom action errors during installation.

```bash
WINEPREFIX=~/.wine-touchgfx winetricks -q vcrun2022
```

---

## 4. Install Windows core fonts

TouchGFX Designer uses WPF typography APIs that call `.First()` on the system font list. If no Windows fonts are installed, this throws `System.InvalidOperationException: Sequence contains no elements`, which propagates to a `NullReferenceException` crash when creating a new project.

```bash
WINEPREFIX=~/.wine-touchgfx winetricks --verbose corefonts
```

This installs Arial, Courier New, Times New Roman, Trebuchet MS, Verdana, and others.

> **Warning:** `winetricks` runs `wineserver -w` after each font installs, waiting for all Wine processes in the prefix to exit. If it hangs there, open a second terminal and run:
> ```bash
> WINEPREFIX=~/.wine-touchgfx wineserver -k
> ```
> Repeat this for each font that hangs. There are ~10 fonts in `corefonts`.

---

## 5. Install TouchGFX Designer

```bash
WINEPREFIX=~/.wine-touchgfx wine msiexec /i ~/path/to/TouchGFX-4.26.1.msi
```

Follow the GUI installer. You may see a harmless dialog error 2726 — this is cosmetic and the install will succeed.

Installed to: `~/.wine-touchgfx/drive_c/TouchGFX/4.26.1/`

---

## 6. Block ST CDN hosts (prevent crash on launch)

TouchGFX Designer tries to download demo thumbnails from `sw-center.st.com` and `riverdi.com` on startup. Under Wine, the file-locking behaviour when these requests fail causes a **NullReferenceException** crash.

Fix: block those domains in the Wine hosts file so the app never attempts the downloads.

```bash
echo "127.0.0.1 sw-center.st.com" >> ~/.wine-touchgfx/drive_c/windows/system32/drivers/etc/hosts
echo "127.0.0.1 riverdi.com"      >> ~/.wine-touchgfx/drive_c/windows/system32/drivers/etc/hosts
```

---

## 7. Clear corrupt download cache

If the Designer has been launched before you applied the hosts fix, corrupt partial PNGs may exist in the cache. Clear them:

```bash
WINEPREFIX=~/.wine-touchgfx wineserver -k
sleep 1
rm -rf ~/.wine-touchgfx/drive_c/users/$USER/AppData/Roaming/TouchGFX-4.26.1/Downloads/
```

---

## 8. Launch TouchGFX Designer

```bash
WINEPREFIX=~/.wine-touchgfx wine \
  ~/.wine-touchgfx/drive_c/TouchGFX/4.26.1/designer/TouchGFXDesigner-4.26.1.exe
```

---

## 9. Headless CLI code generation (no GUI required)

`tgfx.exe` can generate TouchGFX source code from a `.touchgfx` project file without opening the Designer UI. This is ideal for CI/CD pipelines.

Reference: [TouchGFX docs — Generating code from the command line](https://support.touchgfx.com/docs/development/ui-development/working-with-touchgfx/multiple-developers#generating-touchgfx-code-from-the-command-line)

```bash
WINEPREFIX=~/.wine-touchgfx wine \
  ~/.wine-touchgfx/drive_c/TouchGFX/4.26.1/designer/tgfx.exe \
  generate -p /path/to/YourProject.touchgfx
```

> **Tip:** Use the Linux path — Wine translates it automatically. Or use a Windows-style path with `Z:` drive (Wine maps `/` → `Z:\`).

### Windows-style path example

```bash
WINEPREFIX=~/.wine-touchgfx wine \
  ~/.wine-touchgfx/drive_c/TouchGFX/4.26.1/designer/tgfx.exe \
  generate -p 'C:\users\dan\projects\MyProject\MyProject.touchgfx'
```

---

## 10. Convenience aliases

Add to `~/.bashrc` or `~/.zshrc`:

```bash
# TouchGFX Designer GUI
alias touchgfx='WINEPREFIX=~/.wine-touchgfx wine ~/.wine-touchgfx/drive_c/TouchGFX/4.26.1/designer/TouchGFXDesigner-4.26.1.exe'

# TouchGFX CLI code generator (headless)
alias tgfx='WINEPREFIX=~/.wine-touchgfx wine ~/.wine-touchgfx/drive_c/TouchGFX/4.26.1/designer/tgfx.exe'
```

Usage after sourcing:

```bash
touchgfx                                    # Open Designer GUI
tgfx generate -p MyProject.touchgfx        # Generate code headlessly
```

---

## Key paths

| Item | Path |
|------|------|
| Wine prefix | `~/.wine-touchgfx/` |
| Designer exe | `~/.wine-touchgfx/drive_c/TouchGFX/4.26.1/designer/TouchGFXDesigner-4.26.1.exe` |
| CLI exe | `~/.wine-touchgfx/drive_c/TouchGFX/4.26.1/designer/tgfx.exe` |
| Designer log | `~/.wine-touchgfx/drive_c/users/$USER/AppData/Roaming/TouchGFX-4.26.1/TouchGFXDesigner.log` |
| Thumbnail cache | `~/.wine-touchgfx/drive_c/users/$USER/AppData/Roaming/TouchGFX-4.26.1/Downloads/` |
| Wine hosts file | `~/.wine-touchgfx/drive_c/windows/system32/drivers/etc/hosts` |

---

## Troubleshooting

### Designer crashes immediately on the home screen

**Symptom:** `NullReferenceException` / `File has a user-mapped section` in the log.

**Cause:** Wine's mmap implementation doesn't honour Windows file-section semantics. When the app tries to re-open a thumbnail PNG that is still memory-mapped by the image renderer, it throws — resulting in a null bitmap and a crash.

**Fix:** Block the CDN hosts (step 5) and clear the cache (step 6), then relaunch.

### `libgcc_s_dw2-1.dll` missing

Fixed by installing `dotnet48` via winetricks (step 2).

### MSI pipe error during install

Fixed by installing `vcrun2022` via winetricks (step 3).

### MSI dialog error 2726

Cosmetic only — the installation completes successfully. Can be ignored.

### Crashes when creating a new project (`Sequence contains no elements`)

**Symptom:** `System.InvalidOperationException: Sequence contains no elements` in the log, crash in `TextUtils.AddDefaultTypographies`.

**Cause:** WPF's typography system calls `.First()` on the installed font list. With no Windows fonts in the prefix, the list is empty and it throws.

**Fix:** Install `corefonts` via winetricks (step 4).

### `winetricks corefonts` hangs at `wineserver -w`

**Cause:** A leftover Wine process in the prefix is keeping the wineserver alive, so `wineserver -w` blocks indefinitely.

**Fix:** From a second terminal, run `WINEPREFIX=~/.wine-touchgfx wineserver -k` to kill all Wine processes. Repeat for each font that hangs (up to ~10 times).
