# lumen-imageviewer

The image viewer for **LoricaOS**, a capability-based, no-ambient-authority
x86-64 operating system built on the from-scratch
[Aegis](https://github.com/LoricaOS/Aegis) kernel. imageviewer decodes a single
image, renders it with fit / zoom / pan controls, and can drag the file back out
to another window. It is a standalone client of the
[lumen](https://github.com/LoricaOS/lumen) compositor, distributed as a
[herald](https://github.com/LoricaOS/LoricaOS) system package and installed into
the `/apps` bundle tree.

## Where imageviewer fits

LoricaOS is decomposed into independent repositories. imageviewer is a leaf: a
GUI application that talks only to the compositor and the kernel.

| Repo | Role |
|------|------|
| [`LoricaOS/Aegis`](https://github.com/LoricaOS/Aegis) | The kernel. Provides the `AF_UNIX` sockets, `memfd`, the filesystem, and the capability model. |
| [`LoricaOS/lumen`](https://github.com/LoricaOS/lumen) | The compositor / display server. Every window imageviewer draws is a proxy surface owned by lumen. |
| [`LoricaOS/glyph`](https://github.com/LoricaOS/glyph) | The GUI toolkit. Supplies the software renderer (`draw_*`), the `stb_image` wrapper (`glyph_pixbuf_*`), and the **client side of lumen's window protocol** (`lumen_client.h`). |
| `LoricaOS/lumen-imageviewer` | **This repo.** An external Lumen client, the same pattern as the calculator, settings, terminal, and file manager. |

imageviewer does not touch the framebuffer or input devices. It connects to
lumen over `/run/lumen.sock`, receives a shared `memfd` to draw into, and gets a
stream of input events back. In herald terms it declares `depends=lumen`.

## What it does

`main()` (`src/main.c`) connects to lumen, creates a single window titled
"Images", decodes one image, and runs a 100 ms-timeout event loop that renders
on demand.

- **Input image.** `argv[1]` is the absolute path to an image file — the file
  manager passes it on open or drag. The image is decoded once via glyph's
  `stb_image` wrapper, `glyph_pixbuf_load_file`. With no argument (or a failed
  load) the window shows a centered "No image" / "Cannot open" message and Esc
  quits.
- **View model.** The view is a zoom factor plus a pan offset expressed in
  *image-space* pixels (`ox, oy` = the image coordinate drawn at the top-left of
  the image rectangle). The default view is **FIT**: the whole image scaled to
  the client area and centered. Zoom is clamped to `[0.02, 64.0]`; pan is clamped
  so the image can never be dragged entirely off-screen.
- **Keyboard.** Wheel zooms about the cursor (the image point under the pointer
  stays fixed); `+` / `=` and `-` zoom about center; `f` / `0` fit; `1` sets 1:1;
  arrow keys nudge the pan; Esc or a close request quits. Arrows arrive either as
  raw VT sequences or as Lumen's synthetic `0xF1`–`0xF4` codes.
- **Gesture routing.** A left-press is disambiguated by **where it lands**: a
  drag on the image area pans, while a drag on the bottom status chip (the
  filename strip) hands the gesture off to `lumen_drag_start(LUMEN_DND_COPY)` to
  drag the file *out* to another window. This keeps panning and drag-and-drop
  from fighting over the same gesture.
- **Status chip.** The bottom strip shows the basename, pixel dimensions, and the
  current zoom percentage.
- **Window sizing.** Defaults to roughly 900×650, clamped to about three quarters
  of the framebuffer reported via the `LUMEN_FB_W` / `LUMEN_FB_H` environment
  variables, with a 320×240 floor.

## Capabilities

LoricaOS has **no ambient authority**: a process can do nothing except through
capabilities granted by kernel policy at exec time. imageviewer's policy
(`pkg/etc/aegis/caps.d/imageviewer`, installed to `/etc/aegis/caps.d/imageviewer`)
is the baseline service profile:

```
service
```

It needs nothing beyond `service`: it connects to the compositor over the Lumen
socket and reads the one image file passed to it. It holds no `AUTH`, `SETUID`,
`FB`, or network capability — it never touches the framebuffer (the compositor
does that), and it reads only the file it was handed.

## Building

imageviewer builds with a musl cross-compiler against the prebuilt glyph
toolkit. The `Makefile` fetches a pinned toolkit artifact, compiles `src/*.c`
against it, and packs a signed herald package.

```sh
make MUSL_CC=/path/to/musl-gcc HERALD_KEY=/path/to/signing.key
```

- `GLYPH_VERSION` pins the [glyph](https://github.com/LoricaOS/glyph) toolkit
  release fetched by `tools/fetch-glyph.sh` (it unpacks `include/` and `lib/`
  into `toolkit/`).
- `MUSL_CC` is the musl cross-compiler (defaults to `musl-gcc` on `PATH`; the
  only toolchain assumption — point it at an Aegis-native `cc` to build on-device
  in the future).
- `HERALD_KEY` is the ECDSA P-256 signing key for the package.
- The link line pulls all four toolkit archives (`-lcitadel -laudio -lauth
  -lglyph`); only the objects actually referenced are contributed.

Output: `lumen-imageviewer.hpkg` (a `class=system` herald package) +
`lumen-imageviewer.hpkg.sig`.

## Package payload

`lumen-imageviewer.hpkg` is a manifest-first, uncompressed POSIX `ustar` archive
with a detached ECDSA-P256/SHA-256 signature (`tools/pack.sh`). The herald
package **id** (`lumen-imageviewer`) differs from the bundle/exec name
(`imageviewer`), and the package installs an `/apps` binary alongside a cap
policy under `/etc` — so it is a `class=system` package: first-party,
signature-trusted, installed verbatim by herald.

```
manifest                             id=lumen-imageviewer, class=system, depends=lumen
apps/imageviewer/imageviewer         the image viewer binary (stripped)
apps/imageviewer/app.ini             the Lumen app-bundle manifest (name=Images, exec=imageviewer)
etc/aegis/caps.d/imageviewer         its capability policy (service)
```

## Repository layout

```
.
├── Makefile          fetch toolkit → build component.elf → pack the .hpkg
├── VERSION           this component's version
├── GLYPH_VERSION     pinned glyph toolkit artifact version
├── src/
│   └── main.c        connect, decode, fit/zoom/pan, drag-out, event loop
├── pkg/              install-tree skeleton shipped verbatim
│   ├── apps/imageviewer/app.ini      the /apps bundle manifest
│   └── etc/aegis/caps.d/imageviewer  the capability policy
└── tools/
    ├── fetch-glyph.sh   fetch + unpack the pinned glyph toolkit artifact
    └── pack.sh          build + sign lumen-imageviewer.hpkg
```

Build outputs (`component.elf`, `*.hpkg`, `*.hpkg.sig`) and the fetched
`toolkit/` are git-ignored.

## Dependencies

`depends=lumen` — imageviewer is a client of the
[lumen](https://github.com/LoricaOS/lumen) compositor, so installing it pulls
lumen, which in turn ships the desktop fonts every Lumen client inherits. There
is no separate font package.

## Sibling components

- [bastion](https://github.com/LoricaOS/bastion) — display manager / login greeter
- [lumen-sysmon](https://github.com/LoricaOS/lumen-sysmon) — system monitor
- [lumen-netman](https://github.com/LoricaOS/lumen-netman) — network status panel
