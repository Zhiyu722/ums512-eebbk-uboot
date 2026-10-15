# UMS512 EEBBK P21H180 U-Boot 移植进度

## 设备信息

| 项目 | 值 |
|---|---|
| 设备型号 | 步步高 P21H180（EEBBK） |
| SoC | 紫光展锐 UMS512（T610 / SHARKL5PRO） |
| 内核基线 | 原厂 Android 4.14 / 5.4 |
| 面板 | `lcd_ft8201ab_boe_mipi_fhd_bbk_H180`（1280x1752，FT8201AB IC） |
| PMIC | SC2730 |
| 充电 IC | BQ2560X（I2C4，从地址 0x6b） |
| 屏幕背光 | PWM index=2 |
| 存储 | eMMC（无 SD 卡槽） |
| 串口 | 无（bootargs 里有 `disable_uart`） |

## 引导链

BootROM → SPL → U-Boot → Linux 内核

- BootROM：芯片固化，验证 SPL 的 DHTB SHA256
- SPL：已 patch，去除了对 U-Boot 的 RSA 校验（IDA 手动 NOP）
- U-Boot：从 uboot 分区加载，当前可以替换
- 内核：待解决（没 SD 卡槽、没串口）

## 当前进度

| 阶段 | 状态 | 说明 |
|---|---|---|
| SPL 解锁 | 完成 | IDA NOP RSA 校验 |
| U-Boot 编译 | 完成 | aarch64 交叉工具链，U-Boot 2015.07 |
| DHTB 打包 | 完成 | 用原厂 512 字节头 + 692 字节尾部 |
| 设备刷入 | 完成 | spd_dump，可刷回 a3uboot.bin |
| 设备启动 | 完成 | 刷入编译版后能开机 |
| LCD 显示 | 未解决 | 见下文 |
| 内核引导 | 未开始 | 需先解决 LCD 或换方案 |

## 关键技术点

### 1. DHTB 头部格式

- 头部 512 字节（0x200）
- 偏移 0x00："DHTB" magic
- 偏移 0x04：version = 1
- 偏移 0x30：payload 长度（8 字节小端）
- 偏移 0x3c：payload 长度 - 642（照抄原厂，8 字节小端）
- 偏移 0x40：SHA256（留空全零，SPL 不校验）
- 原厂在 payload 之后还有 692 字节尾部（含哈希、对齐），必须保留，否则黑屏
- 打包方式：用原厂 a3uboot.bin 的前 512 字节头 + 原厂尾部 692 字节，中间替换 payload

打包脚本示例：

```python
import struct
with open("a3uboot.bin", "rb") as f:
    orig = f.read()
with open("out/u-boot-dtb.bin", "rb") as f:
    payload = f.read()

header = bytearray(orig[:512])
size = len(payload)
header[0x30:0x38] = size.to_bytes(8, 'little')
header[0x3c:0x44] = (size - 642).to_bytes(8, 'little')

tail = orig[0x200 + 0xd0340:]  # 原厂尾部 692 字节
with open("out/new_uboot.bin", "wb") as f:
    f.write(header)
    f.write(payload)
    f.write(tail)

