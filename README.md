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

## Credits

Encrypted system-dump and `KERNEL_INFO` support is adapted from
[Princess-of-Sleeping's spsp2dmp-parser](https://github.com/Princess-of-Sleeping/spsp2dmp-parser),
specifically commit `d8b5ca8d10c4fc9e286e0d5c810efc5231a2989b`. Her container
definitions, keys, and decryption work made this support possible. The original
MIT copyright notice is preserved in [LICENSE](LICENSE).
