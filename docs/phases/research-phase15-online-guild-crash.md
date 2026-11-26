# Phase 15-A — Online Guild entry crash

On iPad Air (M2) running iPadOS 18.7.3, entering Online Guild triggered a UIKit assertion during a background-thread view hierarchy modification. The crash report recorded `EXC_CRASH` / `SIGABRT` on Thread 2, executable UUID `58cc55c7-60ca-3196-b5d2-0c2aeb64b32b`.

## Call site and correction

Manual Ghidra analysis located the `addSubview:` message send in the function starting at `0x1003DEFE0`. The name `Lobi_Initialize` is a descriptive research label. The Mach-O `__TEXT` segment starts at virtual address `0x100000000`, file offset zero, with 6,389,760 file-backed bytes. The call is in `__TEXT,__text`.

| Step | Unslid address | File offset | Original → replacement (little endian) |
|---|---|---|---|
| `ipad.online_guild.lobi_view_attachment` | `0x1003DF09C` | `0x3DF09C` | `7ede0294` → `1f2003d5` |

Replacing `bl _objc_msgSend` with `nop` skips the view attachment. Execution falls through to `mov w8, #1` and `strb w8, [x22, #0x708]`. The subsequent account-name setup, singleton access, global initialization, and status queries keep their original instructions. This workaround avoids one UIKit call. Broader UIKit thread safety and effects on Lobi chat remain unverified.

## Reproduction

| Artifact | SHA-256 |
|---|---|
| Phase 14 production executable | `ff26548f0e32f240e01c30e2e172bd005087b43ea1a517527bc84668339be10e` |
| Device-tested Phase 15-A executable | `6d7bab9083ed757ea71b2c454a48e2f13476804b71c0f4dc964ff5a72d3d738c` |
| Device-tested experimental IPA | `dd4d7a16d34d8845da252251ae47d010269c005500a5a7488b42ba9d2e397151` |

The original arm64 slice and common patched baseline both contain the expected call bytes. The permanent builder exports seven steps covering 376 bytes. The production profile uses six additional steps covering 372 bytes and retains its 456-byte `safe-area.34` override. The permanent and production executables reproduce the experimental reference across all 8,593,744 bytes. Exactly four bytes differ from Phase 14. The experimental IPA preserves all 14,204 member paths and their order, with the main executable as the only changed member.

## Device observations and scope

With the experimental IPA, Online Guild opened the room selection screen without the original crash. Selecting any room produced a communication error after several seconds. Multiplayer connectivity remains unavailable.

The tester subsequently confirmed that the production-generated IPA works as intended on a physical iPad Air (M2). Phase 15-A production integration and hardware validation are complete. The workaround resolves the original Online Guild entry crash and allows access to room selection. Online multiplayer connectivity remains unresolved.

Static inspection identified GameKit, GKLocalPlayer, GKMatchRequest, GKMatchmaker, GKMatchDelegate, and legacy Lobi integration. Game Center authentication status, the cause of the matchmaking failure, and remote service availability remain unconfirmed. Diagnosing the GameKit matchmaking failure and investigating possible PPSSPP interoperability remain separate areas for future research.
