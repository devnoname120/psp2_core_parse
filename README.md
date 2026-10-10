# psp2-core-parse

Analyze a PlayStation Vita psp2core dump:

```sh
./psp2-core-parse analyze dump.psp2dmp
```

## Encrypted system dumps

Raw ELF, gzip-compressed `.psp2dmp`, and encrypted version-2 `.spsp2dmp`
containers are detected by their contents, not their filename. Interrupted
`.tmp` and `.spsp2tmp` files can be analyzed when enough data survives to recover
an ELF prefix. Encrypted input requires the optional `cryptography` dependency:

```sh
python -m pip install '.[crypto]'
./psp2-core-parse analyze dump.spsp2dmp
./psp2-core-parse info interrupted.spsp2tmp
./psp2-core-parse extract dump.spsp2dmp --elf --output decrypted.psp2dmp
```

The encrypted container's section IDs, offsets, lengths, and IVs are read from
its header. Section 0 is decrypted with AES-128-CBC and then decompressed as
gzip. Other sections are inventoried but are not appended to the gzip stream.
Logical section lengths need not be AES-aligned: the final ciphertext block is
decrypted in full, then the payload is limited to its declared logical length.
Only complete captured AES blocks are decrypted; truncated sections remain
errors, even when the complete ELF can be recovered. `--strict` rejects these
incomplete dumps. A complete container header is required; unsupported versions,
encryption key IDs, and malformed or unfinalized section descriptors are rejected.

**Container HMACs are not verified.** Structural completeness and successful gzip
decompression do not authenticate the dump. This limitation is also reported by
`info`, `analyze`, and their JSON output. Plain ELF/gzip input still works without
installing the optional dependency.

Kernel-crash dumps can contain a separately encrypted `KERNEL_INFO` ELF. Extract
its decrypted segments explicitly:

```sh
./psp2-core-parse extract dump.spsp2dmp --kernel-info --output kernel.elf
```

This supports the reference layout with one wrapped-key `PT_NOTE`: an
AES-128-CBC-wrapped 256-bit key and independently encrypted AES-256-CBC data
segments. The nested ELF and its segments must be complete. Ordinary
`--note KERNEL_INFO` extraction preserves the original note bytes, and `--elf`
preserves the outer ELF, including the still-encrypted nested kernel data.
Neither extraction modifies the input file.

## Usage

Use a matching decrypted Vita ELF for symbols, source locations, disassembly, and unwinding:

```sh
./psp2-core-parse analyze dump.psp2dmp \
  --image application=application.elf
```

The name before `=` must be the actual runtime module name reported by `modules`; use `application` only when that is the module's real name.

VitaSDK must be available through `PATH` or `VITASDK` when using these image-backed features.

Write the complete analysis as JSON:

```sh
./psp2-core-parse analyze dump.psp2dmp --json
```

Other basic commands:

```sh
./psp2-core-parse info dump.psp2dmp
./psp2-core-parse notes dump.psp2dmp
./psp2-core-parse notes dump.psp2dmp THREAD_INFO --raw
./psp2-core-parse threads dump.psp2dmp
./psp2-core-parse thread dump.psp2dmp crash
./psp2-core-parse registers dump.psp2dmp crash
./psp2-core-parse stack dump.psp2dmp crash --bytes 0x200
./psp2-core-parse backtrace dump.psp2dmp crash --image application=application.elf
./psp2-core-parse modules dump.psp2dmp
./psp2-core-parse memory-blocks dump.psp2dmp --captured-only
./psp2-core-parse libraries dump.psp2dmp --nid 0x00000000
./psp2-core-parse files dump.psp2dmp
./psp2-core-parse apps dump.psp2dmp
./psp2-core-parse processes dump.psp2dmp
./psp2-core-parse budgets dump.psp2dmp
./psp2-core-parse events dump.psp2dmp
./psp2-core-parse callbacks dump.psp2dmp
./psp2-core-parse timers dump.psp2dmp
./psp2-core-parse devices dump.psp2dmp
./psp2-core-parse summary dump.psp2dmp
./psp2-core-parse context dump.psp2dmp
./psp2-core-parse system2 dump.psp2dmp --output display.caf
./psp2-core-parse map dump.psp2dmp
./psp2-core-parse address dump.psp2dmp 0x81000000 --image application=application.elf
./psp2-core-parse memory dump.psp2dmp 0x81000000 0x40
./psp2-core-parse xref dump.psp2dmp 0x81000000
./psp2-core-parse search dump.psp2dmp --ascii "example"
./psp2-core-parse disasm dump.psp2dmp --image application=application.elf
./psp2-core-parse waits dump.psp2dmp
./psp2-core-parse object dump.psp2dmp 0x40010001
./psp2-core-parse tty dump.psp2dmp --tail 40
./psp2-core-parse validate dump.psp2dmp
./psp2-core-parse triage *.psp2dmp
./psp2-core-parse compare first.psp2dmp second.psp2dmp
./psp2-core-parse extract dump.psp2dmp --note THREAD_REG_INFO --output note.bin
```

Supply more than one decrypted image when a dump contains multiple modules:

```sh
./psp2-core-parse analyze dump.psp2dmp \
  --image application=application.elf \
  --image plugin=plugin.elf
```

Inspect the crashed thread and resolve its execution state:

```sh
./psp2-core-parse thread dump.psp2dmp crash --image application=application.elf
./psp2-core-parse thread dump.psp2dmp 3
./psp2-core-parse thread dump.psp2dmp 0x40010001
./psp2-core-parse registers dump.psp2dmp crash --json
./psp2-core-parse stack dump.psp2dmp crash --bytes 0x400 --candidates 32 \
  --image application=application.elf
./psp2-core-parse backtrace dump.psp2dmp crash --max-frames 16 \
  --image application=application.elf
./psp2-core-parse disasm dump.psp2dmp 0x81000001 --thumb --bytes 0x80 \
  --image application=application.elf
```

Inspect and search captured memory:

```sh
./psp2-core-parse address dump.psp2dmp 0x81000000 --bytes 0x40
./psp2-core-parse memory dump.psp2dmp 0x81000000 0x100
./psp2-core-parse memory-blocks dump.psp2dmp --address 0x81000000
./psp2-core-parse xref dump.psp2dmp 0x81000000 --limit 64
./psp2-core-parse search dump.psp2dmp --hex 00112233 --limit 64
./psp2-core-parse search dump.psp2dmp --ascii "fatal error" --limit 64
```

Filter supporting records:

```sh
./psp2-core-parse notes dump.psp2dmp MODULE_INFO --raw --json
./psp2-core-parse libraries dump.psp2dmp --name SceExample
./psp2-core-parse libraries dump.psp2dmp --address 0x81000000
./psp2-core-parse files dump.psp2dmp --process 0x10001
./psp2-core-parse budgets dump.psp2dmp --name example
./psp2-core-parse events dump.psp2dmp --title-id TITLE00000
./psp2-core-parse events dump.psp2dmp --flags 0x10000
./psp2-core-parse waits dump.psp2dmp --active
./psp2-core-parse tty dump.psp2dmp --stream TTY_INFO --tail 100
```

Extract raw evidence:

```sh
./psp2-core-parse extract dump.psp2dmp --note THREAD_REG_INFO --output note.bin
./psp2-core-parse extract dump.psp2dmp --load 3 --output load.bin
./psp2-core-parse extract dump.psp2dmp \
  --memory 0x81000000 0x100 --output memory.bin
./psp2-core-parse system2 dump.psp2dmp --output display.caf
```

Triage and compare multiple dumps:

```sh
./psp2-core-parse triage dumps/
./psp2-core-parse triage *.psp2dmp --json
./psp2-core-parse compare first.psp2dmp second.psp2dmp
./psp2-core-parse compare first.psp2dmp second.psp2dmp --json
```

Add `--json` to query commands for machine-readable output. Add `--strict` to reject structurally incomplete dumps.

## Kernel dumps (.psp2kerndmp)

Development-kit kernel panic and exception dumps are also supported. They use a
different container from ELF `.psp2dmp` cores. The CLI detects the format from the
header, including files renamed to another extension; `.psp2kerndmp` also selects
the kernel parser for damaged headers. ELF, gzip, and encrypted system-dump
signatures take precedence over the filename.

Plaintext kernel dumps require no additional dependencies. For encrypted kernel
dumps, install the optional AES backend:

```sh
python3 -m pip install '.[kernel]'
```

Both `0x38`- and `0x40`-byte blob headers are supported, with automatic AES-256-CBC
decryption when the container's encryption flag is set. No separate decryption
or conversion command is needed.

```sh
./psp2-core-parse analyze dump.psp2kerndmp
./psp2-core-parse analyze dump.psp2kerndmp --json
./psp2-core-parse info dump.psp2kerndmp
./psp2-core-parse registers dump.psp2kerndmp
./psp2-core-parse backtrace dump.psp2kerndmp
./psp2-core-parse threads dump.psp2kerndmp
./psp2-core-parse thread dump.psp2kerndmp crash
./psp2-core-parse modules dump.psp2kerndmp
./psp2-core-parse context dump.psp2kerndmp --json
./psp2-core-parse validate dump.psp2kerndmp --strict
./psp2-core-parse extract dump.psp2kerndmp --blob --output kernel-blob.bin
./psp2-core-parse triage dumps/ --json
./psp2-core-parse compare first.psp2kerndmp second.psp2kerndmp
```

The analysis includes firmware/session metadata, process/thread/CPU IDs, system
time, source hashes, separate kernel and user backtraces, exception and saved
thread registers, VFP registers, fault-status decoding, and waypoint records with
module fingerprints/bases. Exception registers and saved thread registers remain
separate: they need not describe the same point in execution. Unknown trailing
blob records are retained in JSON and by `extract --blob`; extracted AES plaintext
is not unpadded. AES-CBC does not authenticate the dump, so structural validation
does not establish authenticity.

Truncated inputs retain complete available records where the headers remain
usable. `--strict` rejects structural damage. Unsupported header layouts, invalid
offsets/sizes, and unknown encryption states produce explicit errors rather than
guessed records.

These dumps are diagnostic snapshots, **not full kernel memory images**. The
module/thread lists contain only evidence present in the saved records, and the
backtraces are producer-saved, not independently unwind-verified. ELF-note,
captured-memory, and image-backed symbolization/disassembly commands are not
available for this format. In particular, `--image` is rejected rather than
silently ignored. Existing ELF-core commands are unchanged. Triage can include
both formats.

For Python callers, `read_dump()` selects the appropriate parser;
`CoreDump.read()` remains the ELF-specific API:

```python
from psp2_core_parse import KernelDump, read_dump

dump = read_dump("dump.psp2kerndmp", strict=True)
if isinstance(dump, KernelDump):
    result = dump.analysis()
```

## Credits

Kernel-dump format definitions and decryption parameters are based on
[Princess of Sleeping](https://github.com/Princess-of-Sleeping)'s
[psp2kerndmp-parser](https://github.com/Princess-of-Sleeping/psp2kerndmp-parser),
reference commit `b18a4ed29a56bfbdd289c749c2200bd9e646520a`.
Her MIT copyright and permission notice is retained in
[`LICENSES/psp2kerndmp-parser.txt`](LICENSES/psp2kerndmp-parser.txt) and included in
the Python distribution.

Encrypted system-dump and `KERNEL_INFO` support is adapted from
[Princess-of-Sleeping's spsp2dmp-parser](https://github.com/Princess-of-Sleeping/spsp2dmp-parser),
specifically commit `d8b5ca8d10c4fc9e286e0d5c810efc5231a2989b`. Her container
definitions, keys, and decryption work made this support possible. The original
MIT copyright notice is preserved in [LICENSE](LICENSE).
