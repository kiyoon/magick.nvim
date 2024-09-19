# magick.nvim

[luarocks magick](https://github.com/leafo/magick) v1.6.0 clone, but re-structured to make it easier to be installed with neovim.

Why? It's [`3rd/image.nvim`](https://github.com/3rd/image.nvim)'s dependency, and the plugin was quite difficult to install on machines without sudo privilege.

You also need ImageMagick v7 shared library and headers.

## Installing [`3rd/image.nvim`](https://github.com/3rd/image.nvim).

### First, install ImageMagick locally

If it's hard for you to use a package manager, run this script to install ImageMagick in `~/.local`.  
This simple downloads the appimage and extracts it to `~/.local`, excluding the `libglib` shared library.

```bash
if ! command -v magick &> /dev/null; then
	echo "ImageMagick could not be found. Installing in ~/.local"
	mkdir -p ~/.local
	curl -s https://api.github.com/repos/ImageMagick/ImageMagick/releases/latest \
		| grep "browser_download_url.*ImageMagick--gcc-x86_64.AppImage" \
		| cut -d : -f 2,3 \
		| tr -d \" \
		| wget -qi - -O magick.appimage
	chmod +x magick.appimage
	./magick.appimage --appimage-extract
	# NOTE: installing libglib will break the system's package manager.
	# Many apps depend on it and it won't work.
	# We need to remove libglib from the extracted AppImage.
	rm squashfs-root/usr/lib/libglib-2.0.so.0
	rsync -a squashfs-root/usr/ ~/.local/
	rm magick.appimage
	rm -rf squashfs-root
else
	echo "ImageMagick found at $(which magick). Skipping installation."
fi
```

### Second, configure dependency of `image.nvim`

Set this repo as a dependency of `image.nvim`. With lazy.nvim:

```lua
  {
    "3rd/image.nvim",
    dependencies = {
      "kiyoon/magick.nvim",
    },
    -- ...
  },
```

## Notes

The original library looks for the appropriate shared library using `pkg-config` but it wasn't able to find the locally installed ones.  
So I modified the `wand/lib.lua` as follows:

```lua
  local proc = io.popen(
    'PKG_CONFIG_PATH="$HOME/.local/lib/pkgconfig:/home/linuxbrew/.linuxbrew/lib/pkgconfig:$PKG_CONFIG_PATH" pkg-config --cflags --libs MagickWand',
    "r"
  )

  --------------------

  local prefixes = {
    "/usr/include/ImageMagick",
    "/usr/local/include/ImageMagick",
    vim.fn.expand "$HOME" .. "/.local/include/ImageMagick",
    "/home/linuxbrew/.linuxbrew/include/ImageMagick",
    -- ...
  }
```