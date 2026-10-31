# zip

ZIP archives for [sysl](https://github.com/sysl-lang/sysl): written and read in memory, stored or
deflated, laid out as PKWARE's APPNOTE.TXT says.

The container — local headers, central directory, end record, and every check a reader owes bytes it
did not write — is sysl, in this repository. Deflate, inflate and CRC-32 come from
[sysl-lang/miniz](https://github.com/sysl-lang/miniz), run raw with storage this package hands them.

```sysl
import sh.sysl.zip.*

var w = writer()

w.add("[Content_Types].xml", "<Types/>".bytes)?
w.add("3D/3dmodel.model", model.bytes)?

val file = w.finish()                      // a Buf[u8]: write it wherever it goes

val zip = open(file.view())?

for e in zip.entries.view()
    print(e.name, e.size, zip.read(e)?.len())
```

## Using it

```hocon
dependencies {
  zip { git = "github.com/sysl-lang/zip", version = "0.1.0" }
}
```

The module is **`sh.sysl.zip`**. miniz comes with it; a program that wants to pass a deflate `Level`
imports `sh.sysl.miniz.Level` as well.

## The surface

```sysl
writer(time: DosTime = EPOCH, level: Level = Level.Default) -> Writer

struct Writer
    time: DosTime                 // stamped on each entry as it is added; change it between adds
    add(*self, name: string, data: []const u8, method: Method = Deflate) -> Result[unit, ZipError]
    finish(self) -> Buf[u8]

open(bytes: []const u8) -> Result[Archive, ZipError]

struct Archive
    entries: Buf[Entry]           // the central directory, in its order
    find(self, name: string) -> Option[Entry]
    read(self, e: Entry) -> Result[Buf[u8], ZipError]

struct Entry
    name: string
    method: Method                // Stored, Deflate, or Other(code)
    crc: u32
    size: usize                   // uncompressed
    packed: usize                 // as stored
    time: DosTime
    flags: u16
    offset: usize                 // of the local header
    is_dir(self) -> bool

dos_time(year, month, day, hour = 0, minute = 0, second = 0) -> Option[DosTime]
check_name(name: string) -> Result[unit, ZipError]
```

`ZipError` implements `Display`, so `print(e)` or `e.to_string()` gives a sentence:
`NotZip`, `BadSignature(at)`, `OutOfRange(at)`, `BadName(name)`, `Mismatch(name)`,
`Unsupported(method)`, `Encrypted(name)`, `Zip64`, `MultiDisk`, `TooLarge`, `TooMany`,
`BadCrc(name)`, `BadSize(name)`, `Corrupt(name)`. It derives `Eq`.

## What the writer does

- **Same calls, same bytes.** Nothing from the clock or the machine goes in. Every entry is stamped
  with the writer's `time`, which starts at 1980-01-01 00:00:00 (the earliest DOS time) and is the
  caller's to set. There are no extra fields.
- **`Deflate` is a request.** Where deflate does not make an entry smaller (an empty file, a few
  bytes, data already compressed) the entry is stored, as `zip` does.
- **Names are UTF-8** and flag bit 11 is always set. The central directory says the archive was made
  on Unix, with mode 0644 for a file and 0755 for a directory (a name ending in `/`). The host
  matters: macOS's Info-ZIP `unzip` reads names from an MS-DOS host as code page 437 whatever bit 11
  says, and turns `café.txt` into `caf+�.txt`.
- **Refused:** a name the reader would refuse (below), a method other than stored or deflate, a
  65,536th entry (`TooMany`), and anything that would take the archive past 4 GiB (`TooLarge`). A
  refused `add` leaves the writer as it was.

## What the reader checks

- **Names that climb out of the extraction directory are refused when the archive is opened**: empty,
  absolute (`/x`, `\x`), a drive letter (`C:x`), any `\` or NUL, any `..` component. `..a` and
  `a..` are names, not components, and are accepted.
- **Every offset and length is checked before it is used**: the end record must end the bytes exactly
  (its comment included), the directory has to fit before it and be filled exactly by its entries,
  each local header has to sit before the directory, and its data has to end there too.
- **Every `read` checks** the local header's signature, that it names the same file as the central
  directory, the CRC-32, and the size. A recorded size deflate could never reach from the stored
  bytes (more than 1032 to 1) is refused before anything is allocated, so four bytes cannot ask for
  four gigabytes.
- **Data descriptors (flag bit 3) are read**: the sizes and CRC come from the central directory, which
  is where they always are.

## What is not here, and why

| absent | why |
|---|---|
| ZIP64 | not needed for the first consumer (a 3MF of three small XML parts); recognised, by its end locator or its extra field, and refused as `Zip64` rather than misread |
| encryption | ZipCrypto is broken and AES is a second format; `Encrypted` |
| methods other than 0 and 8 | bzip2, LZMA, zstd each need a codec; listed with `Other(code)`, refused on `read` |
| split and multi-disk archives | `MultiDisk` |
| writing to or reading from files | the package opens nothing, so it needs no `os`: `sysl.fs.write_bytes(path, w.finish().view())` and `open(read_bytes(path)?)` |
| streaming (entries larger than memory) | everything is in a `Buf`; a streaming writer would need data descriptors and a sink |
| duplicate-name detection | a reader answers the first match from `find`; a writer trusts the caller |
| names in code page 437 | a name that is not UTF-8 is refused as `BadName` |
| archive and entry comments, extra fields | read past, never written or exposed |

## Tests

`sysl test .` — 41 tests, against oracles that are not this code:

- **The system's Info-ZIP `unzip` and `zipinfo`** read archives written here: `unzip -t` finds every
  CRC good, `unzip -p` extracts the bytes that went in, `zipinfo` reports each entry's mode, method,
  size, time and name, and `zipinfo -v` the version needed and the absence of data descriptors.
- **Archives other writers made**, in `tests/fixtures/`: by `zip` (deflated and `-0`, with its extra
  fields and a directory entry) and by Python's `zipfile` (data descriptors from an unseekable sink,
  a UTF-8 name, an archive comment, and `../evil.txt` and `/etc/evil`, both refused).
- **One archive worked out byte by byte** from APPNOTE.TXT, which the writer must reproduce exactly
  and the reader must read.
- **Malformed archives**: every truncation of four archives, a bad CRC, bad sizes, bad signatures,
  offsets out of range, zip-slip names, encrypted, ZIP64, multi-disk, and a fixed-seed fuzz of 3,000
  mangled archives that must never crash. The fuzz is what found the allocation bomb the size check
  now refuses; it has a time limit so that it would find it again.

A mutation pass (18 mutations, one at a time) killed every one; three survived the first round and
each got the test that now catches it. The suite also runs clean under AddressSanitizer
(`SYSL_EXTRA_CFLAGS="-fsanitize=address -g" sysl test .`, `instrumented: address (1 binary)`).

The suite needs `unzip` and `zipinfo` on the path.

## Licence

ISC. See [LICENSE](LICENSE).
