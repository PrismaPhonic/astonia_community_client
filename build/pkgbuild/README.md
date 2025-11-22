# Astonia Client - Arch Linux Package

This directory contains the PKGBUILD for building and installing the Astonia 3 Community Client on Arch Linux.

## Installing

To build and install in one step:

```bash
makepkg -si
```

## Running

After installation, run the client with:

```bash
astonia-client
```

## Dependencies

This package will automatically install:
- sdl3
- sdl2-compat
- sdl2_mixer
- libpng
- libzip
- zlib

## Uninstalling

```bash
sudo pacman -R astonia-client
```

