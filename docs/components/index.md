# Components

`go-coff` is three repositories layered library → CLI → packaging tool.

| Module | Import / install | Job |
|--------|------------------|-----|
| [`peln`](peln.md) | `go get github.com/go-coff/peln` | Pure-Go PE/COFF library: link objects/PIE → PE32+/EFI, append sections (UKI), emit bare-metal images. |
| [`pectl`](pectl.md) | `go install github.com/go-coff/pectl@latest` | Cobra reference CLI over `peln`: `link`, `link-pie`, `objcopy`, `append`, `pack`, `sign`. |
| [`efipack`](efipack.md) | `go get github.com/go-coff/efipack` | Host-side library that compresses a PE32+/EFI binary into a self-extracting PE32+/EFI envelope. |

All three are pure Go (`CGO_ENABLED=0`), BSD-3-Clause licensed, and target
PE32+/EFI machines amd64, arm64, riscv64 and loongarch64.
