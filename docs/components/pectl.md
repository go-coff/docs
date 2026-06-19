# pectl — reference CLI

`github.com/go-coff/pectl` is the Cobra-based reference CLI for building UEFI
PE/COFF binaries in pure Go, built on the [`peln`](peln.md) library. It
**links**, **converts**, **appends to**, **packs** and **signs** PE32+/EFI
images without binutils, LLD, `objcopy` or `sbsign`.

```sh
go install github.com/go-coff/pectl@latest
```

Pre-built binaries for linux/macOS/windows are attached to each release.

## Subcommands

```text
pectl link     [flags] OBJECT...   # link relocatable .o files → PE32+ EFI
pectl link-pie [flags] PIE-ELF     # convert a position-independent ELF → PE32+ EFI
pectl objcopy  [flags] INPUT.elf   # ELF → flat binary / SREC / Intel HEX / U-Boot uImage
pectl append   [flags] INPUT       # add PE sections (UKI assembly)
pectl pack     [flags] INPUT.efi   # compress PE32+ EFI → self-extracting PE32+ EFI
pectl sign     [flags] INPUT       # Authenticode-sign for SecureBoot
```

### `pectl link`

Link one or more COFF/PE or ELF **relocatable** objects (e.g. from TinyGo or
clang) into a self-contained PE32+/EFI application — the pure-Go replacement for
`lld-link /subsystem:efi_application`, covering targets LLD's COFF driver does
not (riscv64, loongarch64). Machine is auto-detected from the first object.

```sh
pectl link --allow-unresolved -o BOOTAA64.EFI main-arm64.o thunk-arm64.o
```

Flags: `--machine` (amd64|arm64|riscv64|loongarch64), `--entry` (default
`_start`), `--subsystem` (default 10), `--image-base`, `--allow-unresolved`,
`--section-alignment`, `--file-alignment`, `--header-reserve`.

### `pectl link-pie`

Convert an already-linked **position-independent** ELF executable (`ET_DYN`, as
produced by `go build -buildmode=pie` for a bare-metal `GOOS` such as
**TamaGo**) into a PE32+/EFI application. Its `R_*_RELATIVE` dynamic relocations
are translated to PE base relocations so UEFI firmware rebases the image at
load; the entry point comes from `e_entry`.

```sh
GOOS=tamago GOARCH=loong64 go build -buildmode=pie -o hello-pie.elf .
pectl link-pie -o BOOTLOONG64.EFI hello-pie.elf
```

### `pectl objcopy`

Convert an ELF executable into a **non-UEFI** bare-metal image — a flat binary
(`objcopy -O binary`), Motorola S-record, Intel HEX, or legacy U-Boot uImage
(`mkimage`) — in pure Go. `-O`/`--output-target` selects `binary` (default),
`srec`, `ihex` or `uimage`; addresses default to the image base / ELF `e_entry`.

```sh
pectl objcopy -O binary -o kernel.img kernel.elf
pectl objcopy -O ihex   -o fw.hex     fw.elf
pectl objcopy -O uimage --load 0x80000 --entry 0x80000 --name linux -o uImage kernel.elf
```

### `pectl append`

Add sections at the end of an existing PE32/PE32+ image while preserving every
existing section byte-for-byte — UEFI Unified Kernel Image (UKI) assembly.

```sh
pectl append --linux=vmlinuz.efi --initrd=initramfs.cpio.gz \
             --cmdline=cmdline --osrel=os-release --uname=uname \
             -o BOOTAA64.EFI stub.efi
```

| Flag        | Section    | Typical contents                       |
| ----------- | ---------- | -------------------------------------- |
| `--linux`   | `.linux`   | kernel image (`vmlinuz` / `bzImage`)   |
| `--initrd`  | `.initrd`  | initramfs (`initramfs.cpio.gz`)        |
| `--cmdline` | `.cmdline` | kernel command line                    |
| `--osrel`   | `.osrel`   | `os-release` metadata                  |
| `--uname`   | `.uname`   | `uname -r` string                      |

For any other section: `--section name=path` (also `-s name=path`), repeatable,
any name up to 8 bytes.

### `pectl pack`

Compress a PE32+/EFI image into a **self-extracting PE32+/EFI envelope**
(decompressor stub + compressed `.payload` section). Architecture is
auto-detected from the input's COFF `Machine` field.

```sh
pectl pack -o BOOTAA64-packed.EFI BOOTAA64.EFI
pectl pack -c flate --level 9 -o out.efi in.efi
```

Flags: `-c`/`--compressor` (`flate` (default) | `lzfse` | `lz4`), `--level`
(default `-1` = `compress/flate.DefaultCompression`), `-o`/`--output`
(required).

- `flate`: stdlib `compress/flate`, runnable envelope.
- `lzfse`: **host-side only** as of M6.2 PR4 — produces a packed PE on disk but
  warns that the embedded runtime stub is still flate-only.
- `lz4`: returns "compressor not implemented in this build".

Runnable envelopes: arm64, riscv64, loong64; amd64 ships the same wire format
but its runtime stub is deferred.

### `pectl sign`

Authenticode-sign a PE/COFF image so SecureBoot firmware whose db trusts the
cert will load it — pure Go, no `sbsign`/`sbsigntools`. Sign **last**: any
post-signing `append` invalidates the Authenticode digest.

```sh
pectl sign --key=db.key --cert=db.crt -o BOOTAA64-signed.EFI BOOTAA64.EFI
```

## Exit codes

| Code | Meaning |
| ---- | ------- |
| `0`  | Success |
| `1`  | Runtime error (bad input, library rejection, write failure) |
| `2`  | Usage error (missing/unknown flag, bad argument, wrong arg count) |

Licensed BSD-3-Clause.
