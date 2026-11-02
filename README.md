# PlayCover JKChess Microphone Fix | 金铲铲之战麦克风修复

**Language / 语言：** Click to expand your preferred language below. / 点击展开需要的语言。

> **Unofficial executable patch / 非官方程序补丁。** Tested with **PlayCover 4.0** (app may still display **3.1.0**) and JKChess **1.14.35.1188** (Res **0.0.35.115**), Mac mini M4, macOS Tahoe 26.5.2. Back up first; other versions are unverified. / 已在上述环境实测，其他版本未经验证，请先备份。

<details open>
<summary><strong>🇺🇸 English — Click to expand / collapse</strong></summary>

# Fix JKChess Microphone Permission Denied on PlayCover (macOS)

> **Unofficial workaround, successfully tested on a Mac mini M4 running macOS Tahoe 26.5.2.** After applying the patch, in-game voice recording, sending, and playback all worked. This modifies the game's executable and may trigger integrity checks or account-related consequences. **It is not guaranteed to work on other game versions.**

## Symptoms

When running **金铲铲之战 (JKChess)** (`com.tencent.jkchess`) through PlayCover, the game displays a **microphone permission denied** message upon entering a room. Voice chat remains unavailable even when microphone access is enabled under **System Settings → Privacy & Security → Microphone**.

## Tested environment

| Item | Tested configuration |
| --- | --- |
| Device | Mac mini (2024), Apple M4, 16 GB RAM |
| macOS | Tahoe 26.5.2 |
| Runtime | **PlayCover 4.0** (installed app may still display **3.1.0**) |
| Game | 金铲铲之战 (JKChess) |
| Game app version | **1.14.35.1188** |
| Game resource version | **0.0.35.115** |
| Bundle ID | `com.tencent.jkchess` |
| Architecture | ARM64 |
| Result | **Voice recording, sending, and playback confirmed working** |

> **PlayCover version display note:** This workaround was tested with the PlayCover **4.0** build used in this setup. After installation, the application may still show **3.1.0** in its version display. This was observed in the tested setup and does not by itself prove that the wrong build was installed. Check the downloaded build/source rather than relying only on the displayed version number. This guide does not establish that all builds labeled 4.0 behave identically.

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

</details>

<details>
<summary><strong>🇨🇳 简体中文 — 点击展开 / 收起</strong></summary>

# PlayCover《金铲铲之战》麦克风权限拒绝修复

> **非官方修复，已在 PlayCover 4.0（界面仍可能显示 3.1.0）、Mac mini M4 / macOS Tahoe 26.5.2 上实测成功。** 修复后可以录制、发送及播放游戏语音。**只适用于与下述特征码和代码逻辑相符的版本**；不保证其他版本有效，也可能触发游戏完整性检查或账号风险。

## 故障现象

通过 PlayCover 运行《金铲铲之战》（`com.tencent.jkchess`），进入房间自动提示麦克风权限被拒绝。即使 macOS「隐私与安全性 → 麦克风」已允许游戏访问，仍无法使用语音。

## 测试环境

| 项目 | 实测配置 |
| --- | --- |
| 设备 | Mac mini (2024)，Apple M4，16 GB 内存 |
| macOS | Tahoe 26.5.2 |
| PlayCover | **4.0**（安装后界面可能仍显示 **3.1.0**） |
| 游戏 | 金铲铲之战（JKChess） |
| 游戏 App 版本 | **1.14.35.1188** |
| 资源版本 | **0.0.35.115** |
| Bundle ID | `com.tencent.jkchess` |
| 架构 | ARM64 |
| 结果 | **语音录制、发送、播放均已验证正常** |

> **关于 PlayCover 版本号：** 本次使用的是 PlayCover **4.0** 构建版本，但安装后软件界面仍可能显示 **3.1.0**。这是本次测试环境中的实际现象，不能仅凭界面显示 `3.1.0` 就断定安装错了版本。建议结合下载来源和构建信息核实；不代表所有标注为 4.0 的构建版本都已验证。

## 最直接的修复方法

**前提：** 已在 macOS 中允许游戏访问麦克风，并在 PlayCover 游戏设置中启用 `checkMicPermissionSync`。**完全退出游戏和 LLDB**，然后在终端执行下面的完整脚本。脚本会验证特征码仅出现一次、备份原程序、修改 4 字节并重新签名。若特征码不匹配，会停止而不修改文件。

```bash
APP="$HOME/Library/Containers/io.playcover.PlayCover/Applications/com.tencent.jkchess.app/jkchess"

python3 - "$APP" <<'PY'
from pathlib import Path
import shutil
import sys

path = Path(sys.argv[1])
if not path.is_file():
    raise SystemExit(f"未找到游戏执行文件：{path}")

pattern = bytes.fromhex("7F 0A 00 71 93 02 88 1A E0 03 13 AA")
replacement = bytes.fromhex("20 00 80 D2")
data = bytearray(path.read_bytes())
count = data.count(pattern)
print("特征码匹配次数:", count)
if count != 1:
    raise SystemExit("匹配次数不是 1：当前版本不适用，未修改。")

offset = data.find(pattern) + 8
backup = path.with_name(path.name + ".mic-backup")
if backup.exists():
    raise SystemExit(f"备份已存在：{backup}；为防止覆盖原始备份，已停止。")

shutil.copy2(path, backup)
data[offset:offset + 4] = replacement
path.write_bytes(data)
print("备份文件:", backup)
print("补丁偏移:", offset)
print("补丁写入完成:", data[offset:offset + 4].hex(" "))
PY

# 修改二进制后重新进行临时签名，保留原有 entitlements
codesign --force --sign - --preserve-metadata=entitlements "$APP"

# 验证签名
codesign --verify --verbose=2 "$APP"
```

若最后验证成功，重新从 PlayCover 启动游戏，进入房间测试录音。**本次实测修复后，录音、发送和播放均正常。**

> 注意：脚本只会检查特征字节是否唯一，并**不能**证明新版本的指令语义与本次一致。对于更新后的游戏，应先反汇编核对，不应盲目运行旧补丁。执行文件修改和临时签名均属于非官方操作。

## 恢复原版（回滚）

如游戏无法启动或希望撤销修改，先完全退出游戏，再运行：

```bash
APP="$HOME/Library/Containers/io.playcover.PlayCover/Applications/com.tencent.jkchess.app/jkchess"
BACKUP="${APP}.mic-backup"

if [ -f "$BACKUP" ]; then
  cp -p "$BACKUP" "$APP"
  codesign --verify --verbose=2 "$APP"
else
  echo "找不到原始备份：$BACKUP"
fi
```

若应用仍无法启动，可使用 PlayCover 重新安装可信来源的游戏包；重新安装可能覆盖本次补丁。

## 技术说明

本次验证环境：

| 项目 | 结果 |
| --- | --- |
| 设备 | Mac mini（2024），Apple M4，16 GB |
| 系统 | macOS Tahoe 26.5.2 |
| 游戏 Bundle ID | `com.tencent.jkchess` |
| 原执行文件大小 | `237807552` 字节 |
| 特征码匹配次数 | `1` |
| 特征码偏移（十进制） | `159258584` |
| 修改偏移（十进制） | `159258592` |
| 原始 4 字节 | `E0 03 13 AA` |
| 替换后 4 字节 | `20 00 80 D2` |

在 ARM64 中，这两条指令分别是 `MOV X0, X19` 和 `MOV X0, #1`。修改会使该位置将 `1` 放入 `X0`，但**没有完整反汇编和上下文，不能确定它具体影响哪一个函数或权限分支**。

排查过程中，macOS 麦克风授权已经开启，游戏进程内 `AVAudioSession.recordPermission` 和 `AVAudioApplication.recordPermission` 均返回 `1735552628`（`'grnt'`，Granted）。接入实体麦克风后故障仍然存在；应用补丁并重新签名后语音恢复。因此，本次现象更符合游戏内部权限判断与 PlayCover 兼容性异常，而非单纯缺少 macOS 授权。

## 注意事项

- **先备份再修改。** 不要删除 `jkchess.mic-backup`。
- **不要将补丁视为通用方案。** 游戏更新后，偏移和逻辑可能变化；即使特征码匹配也需进一步核对。
- **重新签名不是开发商原始签名。** 可能影响完整性检查、更新或账号使用，风险自行评估。
- **不要反复重置已经正常工作的麦克风权限。**
- 若尚未出现系统授权，先在 macOS「隐私与安全性 → 麦克风」确认游戏有权限；此补丁不能代替真实的系统授权。

## 相关问题

- PlayCover 官方问题报告：[Microphone not working in games (cannot access Mac microphone) — #2110](https://github.com/PlayCover/PlayCover/issues/2110)

---

**状态：** 2026-11-03 实测成功。此文档记录一次实际修复案例，不是 PlayCover 或游戏开发商的官方支持方案。

</details>
