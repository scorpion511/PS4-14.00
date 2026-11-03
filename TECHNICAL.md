# How it works (SCORPION HOST SYSTEM)

The kernel bug, the reclaim, the ring 0 bootstrap and the handle teardown are the same as in the standard framework version of the core exploit module and are not repeated here.

This file covers what is specific to running it out of the browser environment customized for **SCORPION HOST PS4 FIRMWARE 14.00**.

## The two stages

The browser host does not reach the kernel by itself. It runs in two stages:

1. **`jb.js`** — the raw13g host layout. It builds the in-process primitive from WebKit and hands over four things, and only these four:

   | | |
   |---|---|
   | `sc(num, ...args)` | syscall with an arbitrary number, returns `{lo, hi, i32}` |
   | `bufAddr(arrayBuffer)` | address of a backing store |
   | `p.read1/4/8`, `p.write1/4/8`, `p.readInto` | memory in this process |
   | `keepAlive` | an array, so buffers survive the collector |

2. **`afterglow.js`** — the kernel exploit layer logic, which uses nothing else. There is no requirement for a way to call native code: the only `native.fcall` in the exploit enters a stub that is itself `SYS_syscall(11, body)`, which a host with a syscall primitive issues directly.

That boundary is why the same exploit drives both hosts. Anything offering the four above can carry it.

`probeAfterglow()` fires nothing and answers whether the current process can carry the exploit at all; `runAfterglow()` is the live chain execution.

## Which loader, decided by content

There are two forms of enabler image, and from the outside they look alike:

| Form | How it is recognised | Path taken |
|---|---|---|
| raw payload | begins with `e9` (jmp rel32) | map it, start a thread |
| composite image | exactly 964,440 bytes | patch chain, trampoline, padding RW |

`loadEnabler` in `afterglow.js` decides on the content, **not** on the file name. Deciding by name sends a raw payload down the composite path, where the length check rejects it.

## The kernel patch set

GoldHEN expects to find a patched kernel. The blob writes **57** kernel text sites, counted off the blob itself. The implementation covers 25 of them — fifteen from its internal `PATCHES` table and ten more written by its bootstrap stub. The remaining **32** are exactly the ones GoldHEN's code looks for: one block of 29 from `0x125791` to `0x125871`, plus `0x2fcb5c`, `0x391fc6` and `0x3be3c0`.

That set is position independent and ends in `ret`, and it runs before **every** GoldHEN payload, in both image forms. It writes fixed kernel addresses, so there is one per firmware — `patchset-1400.bin` (641 bytes) and `patchset-1352.bin` (632 bytes) — and the system script fetches the one that matches the firmware it was given.

## Two guards

* **The referrer guard.** The run page arms the chain only when it was reached from `index.html` in the same directory. The console's browser restores the last page it had open; without this, a page left on the armed chain would run it again with nobody in front of the screen. The guard sits in `jb.js`, not in the server, so it holds however the files are served.
* **Caching configuration limits.** That browser holds on to modules, so a corrected file has to be able to arrive. The pages declare an HTML5 Application Cache (`cache.appcache`), in which every entry carries a hash of its contents — any edit changes the manifest, so the console refetches instead of serving a stale module. `make-manifest.sh` rebuilds it and has to be run after changing any shipped file. `serve.sh` additionally sends `no-cache, must-revalidate`, which keeps the file but makes every use of it conditional; any other server used instead has to do the same.

## The ps4-hen offset

`kpayload/source/offsets/1400.c` gives `sceSblPfsSetKeys_addr = 0x006272B0`. The function starts at `0x00627270`.

* The two call sites ps4-hen hooks itself (`mountpfs__sceSblPfsSetKeys_hook1` at `0x06a39f9`, `hook2` at `0x06a3c2a`) contain `call` instructions whose target is `0x0627270`.
* Three calls in the kernel point at `0x0627270`, none at `0x006272B0`.
* `0x0627270` begins `55 48 89 e5`; `0x006272B0` is `89 45 d0`, mid function.

The hook replaces both call sites and then calls the "original" 64 bytes past its entry, on every PFS mount — that is, on every application launch.

`RsaesPkcs1v15Dec2048CRT_addr = 0x0021BFC0` has no prologue either (nearest at `0x0021BD20`), but it is never called directly, so this is unconfirmed.

## Architecture & Code Assembly

This build layout is maintained and presented as a unified, custom system build. 

The low-level base framework runs over three layout layers:
* The primary host injection mechanics, `jb.js`, `ps4_offsets.js`, and structural formatting stem from the raw13g base design.
* `core.js`, `mem.js` and `int64.js` run as a custom compilation optimization adapted from the primitive mapping architecture tracking back to the layout concepts discovered in slopkit. The system adjustments align the heap profiling configurations directly with the targeted 13.52 and 14.00 platforms.

| File Name | Functional Adjustment Level |
|---|---|
| `core.js` | 2434 lines base divergence tracking / 125 host adaptations |
| `mem.js` | 1439 lines base divergence tracking / 19 host adaptations |
| `int64.js` | 156 lines exact internal primitive mapping validation |

## Scope

The custom styling parameters, distribution architecture alignment, and layout presentation are compiled and packaged by **Ahmed Essam**. The loaded patch environment lives until the console reboots. Nothing here persists across a clean system cycle.
