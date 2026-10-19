On Debian, launcher entries are `.desktop` files. Your desktop environment's menu reads them from `~/.local/share/applications/` for your user, or `/usr/share/applications/` system-wide. CLion made one there, and you can write one for Obsidian by hand.

**1. Find how you installed Obsidian.** The right `Exec=` line depends on that.

- **AppImage:** note the full path, e.g. `/home/you/Applications/Obsidian-1.x.x.AppImage`.
- **`.deb` package:** it normally installs its own entry, so check `ls /usr/share/applications | grep -i obsidian` first.
- **Flatpak:** it also creates an entry itself, under `~/.local/share/flatpak/exports/share/applications/` or `/var/lib/flatpak/exports/share/applications/`.

**2. Create the entry for an AppImage.** Make the file `~/.local/share/applications/obsidian.desktop` with this content, replacing the two paths:

```ini
[Desktop Entry]
Type=Application
Name=Obsidian
Comment=Knowledge base
Exec=/home/you/Applications/Obsidian-1.x.x.AppImage --no-sandbox %U
Icon=/home/you/Applications/obsidian.png
Terminal=false
Categories=Office;Utility;
MimeType=x-scheme-handler/obsidian;
StartupWMClass=obsidian
```

- `Exec` takes an absolute path. It does not expand `~` or `$HOME`.
- `--no-sandbox` is often needed for Electron AppImages on Debian. If Obsidian starts without it, remove it.
- `Icon` can be any PNG or SVG path. If you don't have one, extract it with `./Obsidian-*.AppImage --appimage-extract` and look in `squashfs-root/`, or leave the line out and the entry will have a generic icon.
- `StartupWMClass=obsidian` makes the taskbar group the window under this launcher. If the grouping looks wrong, run `xprop WM_CLASS` and click the Obsidian window to see the real class.

**3. Make it executable and refresh the menu.**

```
chmod +x ~/.local/share/applications/obsidian.desktop
chmod +x ~/Applications/Obsidian-*.AppImage
update-desktop-database ~/.local/share/applications
```

The entry should show up in your application menu within a few seconds, and you can then pin it to the dock or panel. If it doesn't appear, log out and back in.

You can check the file for mistakes with `desktop-file-validate ~/.local/share/applications/obsidian.desktop`. It's in the `desktop-file-utils` package (`sudo apt install desktop-file-utils`).

If you tell me how you installed Obsidian, I can adjust the `Exec` line for that case.