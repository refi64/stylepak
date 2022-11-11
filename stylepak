#!/bin/bash

# This Source Code Form is subject to the terms of the Mozilla Public
# License, v. 2.0. If a copy of the MPL was not distributed with this
# file, You can obtain one at http://mozilla.org/MPL/2.0/.

set -e

GTK_THEME_VER=3.22

[[ -n "$STYLEPAK_VERBOSE" ]] && set -x ||:

usage='usage: stylepak install-system|install-user|clear-cache [<theme>]'

die() {
  echo "$@" >&2
  exit 1
}

cache_home="${XDG_CACHE_HOME:-$HOME/.cache}"
data_home="${XDG_DATA_HOME:-$HOME/.local/share}"
stylepak_cache="$cache_home/stylepak"
repo_dir="$stylepak_cache/repo"

old_cache="$cache_home/pakitheme"
if [[ -e "$old_cache" ]]; then
  # If the cache under the old name still exists, try to preserve it under the new name.
  # But, if the new cache also already exists, just trash the old one instead.
  [[ -e "$stylepak_cache" ]] && rm -rf "$old_cache" || mv "$old_cache" "$stylepak_cache"
fi

install_target=""
theme=""

for arg in "$@"; do
  case "$arg" in
  -h|--help) echo "$usage"; exit ;;
  -*) die "$usage" ;;
  *)
    if [[ -z "$install_target" ]]; then
      case "$arg" in
        install-system) install_target=system ;;
        install-user) install_target=user ;;
        *) die "$usage" ;;
      esac
    elif [[ -z "$theme" ]]; then
      theme="$arg"
    else
      die "$usage"
    fi
    ;;
  esac
done

[[ -n "$install_target" ]] || die "$usage"
[[ -n "$theme" ]] || \
  theme=$(gsettings get org.gnome.desktop.interface gtk-theme | tr -d "'")

echo "$theme" | grep -Eq '^[A-Za-z0-9._\-]+$' || \
  die "Invalid theme name (may only contain letters, digits, '_', '-'): $theme"

echo "Converting theme: $theme"
app_id=org.gtk.Gtk3theme.$theme

for location in "$data_home/themes" "$HOME/.themes" /usr/share/themes; do
  if [[ -d "$location/$theme" ]]; then
    echo "Found theme located at: $location/$theme"
    theme_path="$location/$theme"
    break
  fi
done

[[ -n "$theme_path" ]] || die 'Could not locate theme.'

root_dir="$stylepak_cache/$theme"
repo_dir="$root_dir/repo"

rm -rf "$root_dir" "$repo_dir"
mkdir -p "$repo_dir"
ostree --repo="$repo_dir" init --mode=archive
ostree --repo="$repo_dir" config set core.min-free-space-percent 0

build_dir="$root_dir/build"

rm -rf "$build_dir"
mkdir -p "$build_dir/files"

theme_gtk_version=$(ls -1d "$theme_path"/* 2>/dev/null \
  | grep -Po 'gtk-3\.\K\d+$' \
  | sort -nr \
  | head -1)

[[ -n "$theme_gtk_version" ]] || \
  die "Theme directory did not contain any recognized GTK themes."

cp -a "$theme_path/gtk-3.$theme_gtk_version/"* "$build_dir/files"

mkdir -p "$build_dir/files/share/appdata"
cat >"$build_dir/files/share/appdata/$app_id.appdata.xml" <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<component type="runtime">
  <id>$app_id</id>
  <metadata_license>CC0-1.0</metadata_license>
  <name>$theme GTK Theme</name>
  <summary>$theme (generated via stylepak)</summary>
</component>
EOF

appstream-compose \
  --prefix="$build_dir/files" \
  --basename="$app_id" \
  --origin=flatpak "$app_id"

ostree --repo="$repo_dir" commit -b base --tree=dir="$build_dir"

bundles=()

while read -r arch; do
  bundle="$root_dir/$app_id-$arch.flatpak"

  rm -rf "$build_dir"
  ostree --repo="$repo_dir" checkout -U base "$build_dir"

  read -rd '' metadata <<EOF ||:
[Runtime]
name=$app_id
runtime=$app_id/$arch/$GTK_THEME_VER
sdk=$app_id/$arch/$GTK_THEME_VER
EOF
  # Make sure there is no trailing newline, so xa.metadata doesn't get confused later
  echo -n "$metadata" > "$build_dir/metadata"

  ostree --repo="$repo_dir" commit \
    -b "runtime/$app_id/$arch/$GTK_THEME_VER" \
    --add-metadata-string "xa.metadata=$(cat $build_dir/metadata)" \
    --link-checkout-speedup \
    "$build_dir"
  flatpak build-bundle --runtime --arch="$arch" \
    "$repo_dir" "$bundle" "$app_id" "$GTK_THEME_VER"

  trap 'rm "$bundle"' EXIT

  bundles+=("$bundle")
  # Note: a pipe can't be used because it will mess with subshells and cause the append
  # to bundles to fail.
done < <(flatpak list --runtime --columns=arch:f | sort -u)

for bundle in "${bundles[@]}"; do
  if [ "$install_target" == "system" ]; then
    sudo flatpak install -y --$install_target "$bundle"
  else
    flatpak install -y --$install_target "$bundle"
  fi
done
