# 🪄 magick.nvim

[luarocks magick](https://github.com/leafo/magick) v1.6.0 clone, but re-structured to make it easier to be installed with neovim.

Why? It's [`3rd/image.nvim`](https://github.com/3rd/image.nvim)'s dependency, and the plugin was quite difficult to install on machines without sudo privilege.

> [!NOTE]
> NixOS users may not need this plugin. See https://github.com/kiyoon/magick.nvim/issues/1

## 🛠️ Installation

### First, install ImageMagick

- Mac:
  - `brew install imagemagick`
  - `brew install pkg-config`
- Ubuntu:
  - `sudo apt update && sudo apt install libmagickwand-dev pkgconf`

If it's hard for you to use a package manager (e.g. on Linux servers), run this script to install ImageMagick in `~/.local/share/nvim/magick/`.  
This simply downloads the appimage and extracts it to `~/.local/share/nvim/magick/`, excluding the `libglib` shared library.

```bash
# WARN: Only compatible with x86-64 Linux.
INSTALL_DIR=$(nvim --headless +'lua io.write(vim.fn.stdpath("data"))' +qa)/magick  # ~/.local/share/nvim/magick
mkdir -p "$INSTALL_DIR"
curl -s https://api.github.com/repos/ImageMagick/ImageMagick/releases/latest \
    | grep "browser_download_url.*ImageMagick-.*-gcc-x86_64.AppImage" \
    | cut -d : -f 2,3 \
    | tr -d \" \
    | wget -qi - -O magick.appimage
chmod +x magick.appimage
./magick.appimage --appimage-extract
# NOTE: installing libglib will break the system's package manager.
# Many apps depend on it and it won't work.
# We need to remove libglib from the extracted AppImage.
rm squashfs-root/usr/lib/libglib-2.0.so.0
rsync -a squashfs-root/usr/ "$INSTALL_DIR"/
rm magick.appimage
rm -rf squashfs-root
```

### Then, install this plugin

For example, set this repo as a dependency of `3rd/image.nvim`. With lazy.nvim:

```lua
  {
    "3rd/image.nvim",
    build = false, -- do not build with hererocks
    dependencies = {
      "kiyoon/magick.nvim",
    },
    -- ...
  },
```

Now you can `require("magick")` to use ImageMagick in neovim!

## 📓 Notes

The original library looks for the appropriate shared library using `pkg-config` but it wasn't able to find the locally installed ones.  
So I modified the `wand/lib.lua` as follows:

```lua
-- 1. original pkg-config command now looks for the shared library in nvim data directory
local nvim_data_pkgconfig = vim.fn.stdpath("data") .. "/magick/lib/pkgconfig"
local proc = io.popen(
  [[PKG_CONFIG_PATH="]]
      .. nvim_data_pkgconfig
      .. [[:$HOME/.local/lib/pkgconfig:/home/linuxbrew/.linuxbrew/lib/pkgconfig:$PKG_CONFIG_PATH" pkg-config --cflags --libs MagickWand]],
  "r"
)

--------------------
-- 2. Search for headers in more directories (note specifically, nvim data directory)

local prefixes = {
  vim.fn.stdpath("data") .. "/magick/include/ImageMagick",
  "/usr/include/ImageMagick",
  "/usr/local/include/ImageMagick",
  vim.fn.expand "$HOME" .. "/.local/include/ImageMagick",
  "/home/linuxbrew/.linuxbrew/include/ImageMagick",
  -- ...
}

--------------------
-- 3. Use absolute path (nvim data directory) of the shared library instead of the relative path
-- This avoids users having to set `LD_LIBRARY_PATH` or `DYLD_LIBRARY_PATH` manually.

--- e.g. libMagickWand-7.Q16HDRI.so
local function get_lib_name()
	local lname = get_flags():match("-l(MagickWand[^%s]*)")
	local suffix
	if ffi.os == "OSX" then
		suffix = ".dylib"
	elseif ffi.os == "Windows" then
		suffix = ".dll"
	else
		suffix = ".so"
	end
	return lname and "lib" .. lname .. suffix
end

local lib_name = get_lib_name()
local lib_path_in_nvim_data = lib_name and vim.fn.stdpath("data") .. "/magick/lib/" .. lib_name

lib = try_to_load(lib_path_in_nvim_data, "MagickWand", lib_name)
```
