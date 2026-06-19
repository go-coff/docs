# peln — PE/COFF library

`github.com/go-coff/peln` is the pure-Go PE/COFF library for building **UEFI
applications** without binutils or LLD. Zero third-party dependencies, 100 %
test coverage.

```sh
go get github.com/go-coff/peln
```

It ships three packages:

| Package | Import | Job |
| ------- | ------ | --- |
| `linker` | `github.com/go-coff/peln/linker` | turn relocatable objects (or a PIE) into a PE32+/EFI image — the pure-Go replacement for `lld-link /subsystem:efi_application` |
| `appender` | `github.com/go-coff/peln/appender` | add sections to an existing PE (UKI assembly) — the equivalent of `objcopy --add-section` |
| `fwimg` | `github.com/go-coff/peln/fwimg` | **non-UEFI** bare-metal images: ELF→flat binary (`objcopy -O binary`), Motorola SREC, Intel HEX, and U-Boot uImage (`mkimage`) |

## `linker` — objects → PE/EFI

`linker.Link` merges one or more COFF/PE or ELF **relocatable** objects (`.o`,
`ET_REL` — e.g. from TinyGo or clang) into a self-contained PE32+/EFI
application. The machine is taken from the first object; supported: amd64,
arm64, riscv64, loongarch64.

```go
import "github.com/go-coff/peln/linker"

data, _ := os.ReadFile("main-riscv64.o")
obj, _ := linker.ReadObject(bytes.NewReader(data), "main-riscv64.o")
out, err := linker.Link([]*linker.Object{obj}, linker.LinkOptions{
    AllowUnresolved: true, // zero-fill externals the EFI runtime ignores
})
_ = os.WriteFile("BOOTRISCV64.EFI", out, 0o644)
```

The pipeline is `Resolve → ComputeLayout → ApplyRelocations → emit`, and each
architecture has its own relocation backend (`reloc_x86_64*.go`,
`reloc_aarch64*.go`, `reloc_rv64.go`, `reloc_loongarch64.go`). It is motivated
by riscv64 and loongarch64, which LLD's COFF driver does not support.

### `LinkPIE` — a position-independent ELF → PE/EFI

`linker.LinkPIE` converts an already-linked **position-independent executable**
(`ET_DYN`, as produced by `go build -buildmode=pie` for a bare-metal `GOOS`
such as **TamaGo**) into a PE32+/EFI image. It maps each `PT_LOAD` segment to a
PE section, pre-applies every `R_*_RELATIVE` relocation while recording an
equivalent `IMAGE_REL_BASED_DIR64` base relocation (so UEFI firmware rebases at
load), and takes the entry point from `e_entry`.

```go
elf, _ := os.ReadFile("hello-pie.elf") // GOOS=tamago GOARCH=loong64 -buildmode=pie
out, err := linker.LinkPIE(bytes.NewReader(elf), linker.PIEOptions{})
_ = os.WriteFile("BOOTLOONG64.EFI", out, 0o644)
```

Supported machines: amd64, arm64, riscv64, loongarch64. (`debug/pe` cannot
*read* machine `0x6264`, but the emitted bytes are a valid PE32+ that firmware
and `objdump`/`llvm-readobj` parse.)

## `appender` — add sections to a PE

`appender.Append` adds new sections at the end of an existing PE32/PE32+ image
while leaving every existing section's RVA, file offset and contents
untouched — exactly the constraint for assembling UEFI **Unified Kernel Images**
(UKIs):

```go
out, err := appender.Append(stub, []appender.Section{
    {Name: ".cmdline", Data: cmdlineBytes, Characteristics: appender.DefaultCharacteristics},
    {Name: ".linux",   Data: kernelBytes,  Characteristics: appender.DefaultCharacteristics},
    {Name: ".initrd",  Data: initrdBytes,  Characteristics: appender.DefaultCharacteristics},
})
_ = os.WriteFile("BOOTX64.EFI", out, 0o644)
```

The stub must reserve enough header padding to absorb the new section-table
entries (`SizeOfHeaders` is not grown); all systemd UEFI stubs do.

## `fwimg` — non-UEFI bare-metal images

Not every bare-metal target is UEFI. For images loaded at a fixed address by a
bootloader or ROM, `fwimg` converts an ELF executable into the relevant
flat/firmware format — again with no binutils/`mkimage`/`srec_cat`:

```go
import "github.com/go-coff/peln/fwimg"

flat, base, _ := fwimg.Flatten(bytes.NewReader(elf), fwimg.FlattenOptions{}) // objcopy -O binary
hex  := fwimg.IHEX(uint32(base), flat, 0)              // Intel HEX
srec := fwimg.SREC(uint32(base), flat, "fw", 0, 0)     // Motorola SREC
uimg := fwimg.UImage(flat, fwimg.UImageOptions{        // U-Boot bootm payload
    Load: 0x80000, Entry: 0x80000, Name: "linux", Arch: fwimg.ArchARM64,
})
```

The CLI exposes this as `pectl objcopy -O binary|srec|ihex|uimage`.

A reference CLI lives in [`pectl`](pectl.md). Licensed BSD-3-Clause.
