# efipack — self-extracting EFI

`github.com/go-coff/efipack` is a pure-Go library that compresses PE32+/EFI
binaries into a **self-extracting** PE32+/EFI image — the UPX-equivalent that
does not otherwise exist for this format. It was designed to mitigate the EDK2
OVMF amd64 `CpuPageTableLib` `#GP` that fires on `LoadImage` / `StartImage` of
sufficiently large EFI binaries (cloud-boot M6.2 milestone).

```sh
go get github.com/go-coff/efipack
```

!!! note "Runnable with `Flate`; `LZFSE`/`LZ4` are host-side only"
    `Pack` embeds a real per-arch self-extracting runtime stub — the old
    `TODO_STUB` placeholder is gone. A **`Flate`-packed** binary is a genuinely
    runnable self-extracting EFI on arm64, riscv64 and loong64 (the amd64
    runtime stub is deferred). `LZFSE` and `LZ4` compress and round-trip
    correctly on the host and assemble a valid packed PE, but the embedded
    stubs decode `FLAT` only, so an `LZFSE`/`LZ4`-packed image does not yet boot
    under firmware — use `Flate` for a runnable packed EFI.

## API surface

```go
import "github.com/go-coff/efipack"

// Detect the architecture of an input PE32+ from its COFF header.
arch, err := efipack.InferArch(peBytes)
// arch ∈ {AmdArch, ArmArch, RiscvArch, LoongArch}

in, _  := os.Open("BOOTX64.EFI")
out, _ := os.Create("BOOTX64-packed.EFI")
res, err := efipack.Pack(in, out, efipack.Options{
    Compressor: efipack.Flate, // default; zero stub cost
    Level:      0,             // 0 → codec default
})
// res.OriginalSize, res.CompressedSize, res.PackedSize, res.Arch, res.Compressor
```

| Type / constant | Purpose |
| --- | --- |
| `Pack(in, out, opts) (PackResult, error)` | host-side compress + envelope |
| `Options{Compressor, Level}` | knobs; zero value = Flate at default level |
| `PackResult{OriginalSize, CompressedSize, PackedSize, Compressor, Arch}` | summary |
| `Compressor` (`Flate` / `LZFSE` / `LZ4`) | algorithm switch; all three host codecs are wired |
| `Arch` (`AmdArch` / `ArmArch` / `RiscvArch` / `LoongArch`) | PE machine |
| `InferArch(pe) (Arch, error)` | read COFF.Machine without `debug/pe` (works on loong64) |
| `ReadPayload(pe) (algo, uncompressedSize, body, err)` | inverse of `Pack`'s envelope |
| `ErrCompressorNotImplemented` | reserved sentinel; no codec returns it (all three host codecs are wired) |

The envelope is a standard PE32+/EFI structure: DOS stub + PE signature + COFF +
optional header + section table + a real per-arch runtime decompressor `.stub` +
a `.payload` body.

## `.payload` wire format

PE/COFF — and this envelope — are **little-endian**:

```text
.payload section body:
  magic         [4]byte   "CBP0"   — cloud-boot pack v0
  algo          [4]byte   "FLAT" | "LZFS" | "LZ4 "
  uncompressed  uint64    little-endian — host-input size in bytes
  compressed    uint64    little-endian — body size in bytes
  body          [N]byte   exactly N=compressed bytes; codec-specific stream
```

The runtime stub reads this header, allocates exactly the right number
of `EfiBootServicesCode` pages, decompresses, then chain-loads via
`gBS->LoadImage` + `gBS->StartImage`.

## Why Flate as the default?

Cloud-boot binaries already link `compress/gzip`, so a Flate-based stub adds
**zero bytes** versus LZFSE's ~100–200 KiB cost. The ~1-point raw-ratio gap
(LZFSE 40.13 % vs Flate 39.11 %) is negligible against the multi-MiB binaries
being compressed. `Flate` is therefore the zero-value default; `LZFSE` and
`LZ4` are host-side only — both produce a valid packed PE, but the embedded
stubs decode `FLAT` only for now.

Licensed BSD-3-Clause.
