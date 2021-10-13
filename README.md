# stylepak

Automatically install your host GTK+ theme as a Flatpak.

## Requirements

In addition to bash and coreutils (which should be available out of the box on most distros),
the following binaries are needed:

- `ostree`
- `appstream-util`

The packages that provide these vary by distro:

- Fedora Silverblue: These packages should be available out of the box.
- Fedora: `dnf install ostree libappstream-glib`
- Debian/Ubuntu and variants: `apt install ostree appstream-util`
- Arch/Manjaro: `pacman -S ostree appstream-glib`
- OpenSUSE: `zypper install libostree appstream-glib`

## Usage

Run `stylepak install-system` and `stylepak install-user` to grab your current GTK+ theme and
install it as a Flatpak into either the system or user installation, respectively. You can also
pass a theme name, e.g. `stylepak install-user Adapta`, and stylepak will install that theme
instead of your current one.

Use `stylepak clear-cache` to clear the theme storage cache.
