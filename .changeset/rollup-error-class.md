---
"@tuler/node-libcmt": patch
---

Export a `RollupError` class for failures raised by the libcmt binding. Failed
libcmt calls now throw a `RollupError` (instead of a plain `Error`) carrying the
negative `errno` and the failed call name in `syscall`. Argument validation
still throws `TypeError`/`RangeError`.
