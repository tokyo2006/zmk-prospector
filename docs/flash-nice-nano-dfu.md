# nice!nano 接收器固件刷写（串口 DFU）

`eyelash_sofle_prospector_dongle_nice_nano.uf2` 约 1.1 MB。在 macOS 上把它拖进 `NICENANO` 盘经常写不完整：盘符不消失，重新插拔后板子没有任何反应。小文件（如 `settings_reset_nice_nano.uf2`）拖拽正常。

大固件改用 bootloader 自带的串口 DFU 刷写。

适用 bootloader：`INFO_UF2.TXT` 显示 `SoftDevice: S140 version 6.1.1`（nice!nano v2 默认）。

## 1. 安装 adafruit-nrfutil（只需一次）

```sh
python3 -m venv ~/.venvs/nrfutil
~/.venvs/nrfutil/bin/pip install adafruit-nrfutil intelhex
```

## 2. 下载固件

从 GitHub Actions 对应构建的 Artifacts 下载 `firmware`，解压得到 `eyelash_sofle_prospector_dongle_nice_nano.uf2`。

或者用 gh：

```sh
gh run download <run-id> -R tokyo2006/zmk-prospector -D fw
```

## 3. UF2 转 DFU 包

```sh
cd fw/firmware
NAME=eyelash_sofle_prospector_dongle_nice_nano

~/.venvs/nrfutil/bin/python - "$NAME" <<'EOF'
import struct, sys
from intelhex import IntelHex
name = sys.argv[1]
data = open(f"{name}.uf2", "rb").read()
ih = IntelHex()
for i in range(0, len(data), 512):
    _, _, _, addr, size, _, _, _ = struct.unpack("<8I", data[i:i + 32])
    ih.frombytes(data[i + 32:i + 32 + size], offset=addr)
ih.tofile(f"{name}.hex", format="hex")
EOF

~/.venvs/nrfutil/bin/adafruit-nrfutil dfu genpkg \
  --dev-type 0x0052 --sd-req 0x00B6 \
  --application "$NAME.hex" "$NAME.zip"
```

`--sd-req 0x00B6` 对应 S140 6.1.1。

## 4. 进入 bootloader 并刷写

1. 快速双击 RST（RST 短接 GND 两次），出现 `NICENANO` 盘即进入 bootloader。
2. 刷写：

```sh
~/.venvs/nrfutil/bin/adafruit-nrfutil --verbose dfu serial \
  --package "$NAME.zip" \
  -p "$(ls /dev/cu.usbmodem* | head -1)" \
  -b 115200 --singlebank
```

输出 `Device programmed.` 即成功，板子会自动重启。

## 5. 确认

```sh
system_profiler SPUSBHostDataType | grep E_Sofle
```

能看到 `E_Sofle-Pro` 说明固件已运行。

## 常见问题

- **找不到 `/dev/cu.usbmodem*`**：没有进入 bootloader，重新双击 RST；或者换一根能传数据的线，直连电脑不要经过 hub。
- **刷完 USB 能识别但屏幕不亮**：先量屏幕模块上 VCC 对 GND 的电压，排查供电和接线。
- **设置异常**：先按 3、4 步刷 `settings_reset_nice_nano.uf2`（这个小文件直接拖进 `NICENANO` 盘也可以），再刷接收器固件。
