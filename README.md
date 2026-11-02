# Fix JKChess Microphone Permission Denied on PlayCover (macOS)

> **Unofficial workaround, successfully tested on a Mac mini M4 running macOS Tahoe 26.5.2.** After applying the patch, in-game voice recording, sending, and playback all worked. This modifies the game's executable and may trigger integrity checks or account-related consequences. **It is not guaranteed to work on other game versions.**

## Symptoms

When running **金铲铲之战 (JKChess)** (`com.tencent.jkchess`) through PlayCover, the game displays a **microphone permission denied** message upon entering a room. Voice chat remains unavailable even when microphone access is enabled under **System Settings → Privacy & Security → Microphone**.

## Tested environment

| Item | Tested configuration |
| --- | --- |
| Device | Mac mini (2024), Apple M4, 16 GB RAM |
| macOS | Tahoe 26.5.2 |
| Runtime | PlayCover |
| Game | 金铲铲之战 (JKChess) |
| Game app version | **1.14.35.1188** |
| Game resource version | **0.0.35.115** |
| Bundle ID | `com.tencent.jkchess` |
| Architecture | ARM64 |
| Result | **Voice recording, sending, and playback confirmed working** |

## Quick fix

**Before proceeding:**

1. Grant JKChess microphone access in macOS System Settings.
2. Enable **`checkMicPermissionSync`** in the game's PlayCover settings.
3. **Fully quit the game and detach/exit LLDB**, if you have been debugging it.
4. Understand that this is an **unofficial binary patch**. It changes the game executable and its code signature, potentially affecting game integrity checks, updates, or your account.

Paste the entire block below into **Terminal**. The script verifies that the expected byte pattern occurs **exactly once**, saves a backup, replaces four bytes, and re-signs the executable. If the pattern does not match, the script stops without patching.

```bash
APP="$HOME/Library/Containers/io.playcover.PlayCover/Applications/com.tencent.jkchess.app/jkchess"

# Patch only if the expected byte pattern is found exactly once.
python3 - "$APP" <<'PY'
from pathlib import Path
import shutil
import sys

path = Path(sys.argv[1])
if not path.is_file():
    raise SystemExit(f"Game executable not found: {path}")

pattern = bytes.fromhex("7F 0A 00 71 93 02 88 1A E0 03 13 AA")
replacement = bytes.fromhex("20 00 80 D2")
data = bytearray(path.read_bytes())
count = data.count(pattern)
print("Pattern matches:", count)
if count != 1:
    raise SystemExit("Expected exactly one match. Aborting without changes.")

offset = data.find(pattern) + 8
backup = path.with_name(path.name + ".mic-backup")
if backup.exists():
    raise SystemExit(f"Backup already exists: {backup}. Aborting to preserve it.")

shutil.copy2(path, backup)
data[offset:offset + 4] = replacement
path.write_bytes(data)
print("Original executable backed up to:", backup)
print("Patch offset:", offset)
print("Patched bytes:", data[offset:offset + 4].hex(" "))
PY

# Re-sign the modified executable with an ad-hoc signature.
# Preserve its existing entitlements.
codesign --force --sign - --preserve-metadata=entitlements "$APP"

# Verify the resulting signature.
codesign --verify --verbose=2 "$APP"
```

**Important:** If the Python step reports an error or aborts, **do not proceed with the `codesign` commands**. The commands are shown together for convenience but must be treated as sequential steps.

If verification succeeds, launch JKChess again from PlayCover, enter a room, and test voice recording. In the tested environment, voice messages could be recorded, sent, and played back successfully.

> **Version warning:** A unique byte-pattern match is only a basic guard. It does **not** establish that the patch is semantically safe on a newer executable. For other versions, inspect the surrounding ARM64 instructions and control flow before applying any modification.

## Undo the patch (restore the original executable)

Fully quit the game, then run:

```bash
APP="$HOME/Library/Containers/io.playcover.PlayCover/Applications/com.tencent.jkchess.app/jkchess"
BACKUP="${APP}.mic-backup"

if [ -f "$BACKUP" ]; then
  cp -p "$BACKUP" "$APP"
  codesign --verify --verbose=2 "$APP"
else
  echo "Original backup not found: $BACKUP"
fi
```

If the application still fails to start, reinstall the game from a trusted source through PlayCover. Reinstallation may overwrite the patch.

## Technical details

The following values were observed for the **tested executable only**:

| Property | Observed value |
| --- | --- |
| Executable size | `237807552` bytes |
| Pattern | `7F 0A 00 71 93 02 88 1A E0 03 13 AA` |
| Pattern match count | `1` |
| Pattern offset (decimal) | `159258584` |
| Patched instruction offset (decimal) | `159258592` |
| Original instruction bytes | `E0 03 13 AA` |
| Replacement instruction bytes | `20 00 80 D2` |

On ARM64, the instructions decode as:

| Before | After |
| --- | --- |
| `MOV X0, X19` | `MOV X0, #1` |

The patch forces the value `1` into register `X0` at that location instead of copying the value from `X19`. **Without a full disassembly and call-context analysis, the exact function or permission branch affected cannot be established.** The observed result is that voice functionality started working after patching and re-signing.

### What was ruled out during troubleshooting?

- macOS had already granted microphone access to JKChess.
- Within the running game process, both `AVAudioSession.recordPermission` and `AVAudioApplication.recordPermission` returned `1735552628` (`'grnt'`, meaning **Granted**).
- Connecting and selecting a physical microphone did **not** resolve the error before patching.
- After the patch, voice recording, sending, and playback worked.

These observations suggest an incompatibility in the game's internal microphone-permission handling under PlayCover, rather than a straightforward missing macOS permission. They do **not** conclusively identify the underlying function responsible.

## Warnings and limitations

- **Keep the backup:** `jkchess.mic-backup` is your copy of the original executable.
- **Not a universal patch:** Game updates can change instruction offsets and surrounding logic. Do not blindly reuse this patch on another version, even if the pattern matches.
- **Code-signing implications:** Ad-hoc signing replaces the original executable signature; integrity checks or updates may be affected.
- **Account risk:** Modifying a game executable may conflict with game rules or trigger integrity detection. Proceed at your own risk.
- **Actual OS permission is still required:** The patch does not replace granting microphone access in macOS.
- **No affiliation:** This guide is not endorsed by PlayCover or the game publisher.

## Related issue

- [PlayCover issue #2110 — microphone not working in games](https://github.com/PlayCover/PlayCover/issues/2110)

---

**Status:** Successfully tested on **November 3, 2026**, with **JKChess app version 1.14.35.1188** and **resource version 0.0.35.115**. This document records one reproducible successful workaround, not an official fix.
