# PlayCover《金铲铲之战》麦克风权限拒绝修复

> **非官方修复，已在 Mac mini M4 / macOS Tahoe 26.5.2 上实测成功。** 修复后可以录制、发送及播放游戏语音。**只适用于与下述特征码和代码逻辑相符的版本**；不保证其他版本有效，也可能触发游戏完整性检查或账号风险。

## 故障现象

通过 PlayCover 运行《金铲铲之战》（`com.tencent.jkchess`），进入房间自动提示麦克风权限被拒绝。即使 macOS「隐私与安全性 → 麦克风」已允许游戏访问，仍无法使用语音。

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
