# lumen-imageviewer

The image viewer for **AspisOS**, a capability-based, no-ambient-authority
operating system built on the from-scratch
[Aegis](https://github.com/AspisOS/Aegis) kernel.

imageviewer is a standalone client of the [lumen](https://github.com/AspisOS/lumen)
compositor, speaking the Lumen external window protocol (the same pattern as
calculator, settings, terminal and filemanager). It is a component of the Lumen
desktop, distributed as a [herald](https://github.com/AspisOS/AspisOS) package
and installed into the `/apps` bundle tree.

## Role in the system

- Decodes a single image via libglyph's `stb_image` wrapper
  (`glyph_pixbuf_load_file`) and renders it into its window with fit / zoom /
  pan controls.
- `argv[1]` is the absolute path to an image file (the file manager passes it on
  open or drag). With no argument it shows a "No image" message and Esc quits.
- View model: a zoom factor plus a pan offset expressed in image-space pixels.
  The default view is FIT — the whole image scaled to the client area, centered.
- Input: mouse wheel zooms about the cursor; `+`/`=` and `-` zoom about center;
  `f`/`0` fit; `1` sets 1:1; arrow keys nudge the pan; Esc or the close request
  quits.
- Gesture routing is split by where a left-press lands: a drag on the image area
  pans, while a drag on the bottom status chip hands off to `lumen_drag_start`
  (`LUMEN_DND_COPY`) to drag the file out to another window.
- The window defaults to roughly 900x650, clamped to about three quarters of the
  framebuffer reported via `LUMEN_FB_W` / `LUMEN_FB_H`.

## Capabilities

imageviewer's cap policy (`pkg/etc/aegis/caps.d/imageviewer`) is the baseline:

```
service
```

It needs no elevated capability beyond the default service profile: it connects
to the compositor over the Lumen socket and reads an image file passed to it. It
holds no AUTH, SETUID, FB or network capability.

Although its herald package id (`lumen-imageviewer`) differs from the bundle/exec
name (`imageviewer`) and it installs an `/apps` binary plus a cap policy, that
naming and cross-tree install make it a `class=system` package: first-party and
signature-trusted, installed verbatim by herald.

## Building

imageviewer fetches a pinned [glyph](https://github.com/AspisOS/glyph) toolkit
artifact (the GUI libraries it links: glyph + libaudio + libauth) and builds
against it, then packs a signed herald package.

```sh
make MUSL_CC=/path/to/musl-gcc HERALD_KEY=/path/to/signing.key
```

- `GLYPH_VERSION` pins the toolkit release fetched by `tools/fetch-glyph.sh`.
- `MUSL_CC` is the musl cross-compiler (the only toolchain assumption — point it
  at an Aegis-native `cc` to build on-device in the future).
- `HERALD_KEY` signs the `.hpkg`.

Output: `lumen-imageviewer.hpkg` (a `class=system` herald package) +
`lumen-imageviewer.hpkg.sig`.

## Package payload

```
/apps/imageviewer/imageviewer        the image viewer binary
/apps/imageviewer/app.ini            the Lumen app-bundle manifest (name=Images, exec=imageviewer)
/etc/aegis/caps.d/imageviewer        its capability policy (service)
```

## Repository layout

```
src/        imageviewer source
pkg/        install-tree skeleton shipped verbatim (apps/ bundle + caps.d)
tools/      fetch-glyph.sh (toolkit fetch) + pack.sh (build the signed .hpkg)
Makefile    fetch toolkit -> build -> pack
VERSION         this component's version
GLYPH_VERSION   the pinned glyph toolkit version it builds against
```

## Dependencies

`depends=lumen` — imageviewer is a client of the
[lumen](https://github.com/AspisOS/lumen) compositor, so installing it pulls
lumen (which in turn provides the desktop fonts).

## Sibling components

- [bastion](https://github.com/AspisOS/bastion) — display manager / login greeter
- [lumen-sysmon](https://github.com/AspisOS/lumen-sysmon) — system monitor
- [lumen-netman](https://github.com/AspisOS/lumen-netman) — network manager
