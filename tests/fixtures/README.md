<!--
Copyright (c) 2026 devnoname120
SPDX-License-Identifier: MIT
-->

# Kernel-dump fixture

`kernel_panic.encrypted.psp2kerndmp` is synthetic, not a device capture. Its
plaintext is `make_blob(0x40, 1) + b"\x10" * 16` from `test_kerndump.py`: a
0x40-byte panic header, one kernel frame, one user frame, saved thread registers,
and a padding-like final block.

The blob was independently encrypted using the OpenSSL CLI with AES-256-CBC,
`-nopad -nosalt`, and the key/IV published in Princess of Sleeping's
[psp2kerndmp-parser](https://github.com/Princess-of-Sleeping/psp2kerndmp-parser).
The container has the encrypted flag (`0x20000000`) and a content-relative blob
offset of `0x28`. Tests compare every decrypted byte, including the final block,
without generating ciphertext through the implementation being tested.

`kernel_log.encrypted.psp2kerndmp` is also entirely synthetic. Its plaintext is
`b"[synthetic] kernel log fragment\n".ljust(0xB0, b"\x00")`, encrypted independently
with the same OpenSSL command and public key/IV. The container records type 1,
firmware `0x03600011`, flags `0x20000001`, and a blob offset of `0x28`; its session
ID is `bytes(range(16))`, not a device identifier. It reproduces an encrypted
container whose payload has no recognizable diagnostic header. Tests require
lossless non-strict extraction without fabricated records, while strict
validation must continue to fail. No private crash captures are included.
