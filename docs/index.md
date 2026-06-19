# go-coff

Pure-Go **PE/COFF tooling for UEFI** — build, transform and compress PE32+/EFI
images without `binutils`, `lld-link`, `objcopy`, `mkimage` or `sbsign`. No cgo,
no third-party dependencies.

PE/COFF is the executable format UEFI firmware loads. `go-coff` produces and
manipulates those images entirely in Go, which removes a host build-time
dependency on a native toolchain and — crucially — covers the targets LLD's COFF
driver does **not** support: **riscv64** and **loongarch64**. (PE/COFF integers
are little-endian.)

The org is layered as a library, a CLI on top of it, and a focused packaging
tool:

- **[`peln`](components/peln.md)** is the library: link relocatable objects (or
  a position-independent ELF) into a PE32+/EFI image, append sections to an
  existing PE (UKI assembly), and emit non-UEFI bare-metal images
  (flat/SREC/IHEX/uImage).
- **[`pectl`](components/pectl.md)** is the Cobra reference CLI built on `peln`:
  `link`, `link-pie`, `objcopy`, `append`, `pack` and `sign`.
- **[`efipack`](components/efipack.md)** compresses a PE32+/EFI binary into a
  self-extracting PE32+/EFI image — the UPX-equivalent that does not otherwise
  exist for this format.

## Components

| Module | Layer | What it does |
|--------|-------|--------------|
| [`peln`](components/peln.md) | library | Link objects/PIE → PE32+/EFI, append sections (UKI), and emit bare-metal images (`linker` + `appender` + `fwimg`). |
| [`pectl`](components/pectl.md) | CLI | Reference command-line front-end over `peln` — `link`, `link-pie`, `objcopy`, `append`, `pack`, `sign`. |
| [`efipack`](components/efipack.md) | packaging | Compress a PE32+/EFI binary into a self-extracting PE32+/EFI envelope (host-side library). |

## Why not call binutils / LLD?

To remove a host build-time dependency on a native linker for Go tools that
assemble UEFI images (installers, boot managers, embedded pipelines), and to
cover **riscv64** and **loongarch64**, which LLD's COFF driver doesn't. The
`go-coff` libraries are pure Go, so they cross-compile anywhere Go does —
including hosts that don't ship binutils (macOS, minimal Alpine).
