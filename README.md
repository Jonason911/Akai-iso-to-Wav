# akai

把 Akai 采样盘的 CD 镜像一键解成 WAV。镜像丢桌面，终端输入 `akai`，回车，拿采样。

```bash
akai
```

---

## 为什么需要它

你手上有一张老采样盘镜像（Akai S1000/S3000、MPC、E-mu 等），然后遇到这些：

- macOS 双击挂不上，磁盘工具提示无法识别
- 解压软件打不开，报"不是有效的归档"
- 转换工具报 `Unknown Format` / `Unsupported ISO disk format`

**这些通常不是镜像坏了。** Akai CD-ROM 用的不是 ISO9660 文件系统，而是自家那套按 512 字节块组织的 partition / volume 结构。多数通用工具只认 ISO9660，所以一律判死。

`akai` 的做法是：先看镜像到底是什么封装，再决定要不要转换、交给哪个提取器。

---

## 特性

- **自动判定封装**：靠扇区尺寸和 `CD001` 签名位置区分三种形态，裸扇区镜像自动转 2048
- **双提取器**：优先 [smpl-extract](https://github.com/DriftwoodAudio/smpl-extract)（自动合并 L/R、保留 loop 点和 root key），失败自动退回 akaiextract
- **智能声道处理**：检测残留的 `-L` / `-R` 单声道对，自动合并成立体声，保留原目录结构
- **依赖自检**：缺工具时自动尝试安装
- **输出报告**：WAV 数量、声道分布、占用空间、顶层分组一目了然

---

## 环境要求

| 项目 | 说明 |
| --- | --- |
| 系统 | macOS / Linux（自动探测桌面路径，macOS 兼容 iCloud 同步） |
| Python | 3.8+（脚本只用标准库做检测与合并） |
| smpl-extract | **必装**，主提取器 |
| libgig | 可选，提供 `akaiextract` 作为备用 |

```bash
# 主提取器
python3 -m pip install --user smpl-extract

# 备用提取器（macOS）
brew install libgig
```

两个都没有时脚本会尝试自动安装 smpl-extract，装不上再试 Homebrew 的 libgig。

---

## 安装

```bash
mkdir -p ~/bin
cp akai ~/bin/akai
chmod +x ~/bin/akai
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

全局安装（不用配 PATH）：

```bash
sudo cp akai /usr/local/bin/akai
sudo chmod +x /usr/local/bin/akai
```

验证：

```bash
which akai
```

装好后下载目录里的原件可以删，真正被调用的是 `~/bin/akai` 那份。

---

## 使用

1. 把镜像放到桌面，支持 `.iso` / `.img` / `.cdr` / `.bin`
2. 打开终端，输入 `akai`
3. 等它跑完，结果在桌面

```
输出：
  <镜像名>_wav/          未做声道合并时的结果
  <镜像名>_wav_stereo/   做了 L/R 合并时的结果（以报告为准）
```

桌面上有多个镜像时会列出编号让你选，直接回车默认选第 1 个。

输出示例：

```
⑥ 完成
  镜像       : akai_native（505 MB）
  提取工具   : smpl_extract
  输出目录   : /Users/you/Desktop/24_wav
  WAV 总数   : 140
    单声道    : 81 个
    立体声    : 59 个
  占用空间   : 143 MB
  顶层分组   : A, B, C
```

---

## 判定原理

两个信号：文件大小对扇区尺寸取余，加 `CD001` 签名出现的位置。

| 检测结果 | 结论 | 处理 |
| --- | --- | --- |
| `CD001` @ 32769，大小能被 2048 整除 | 标准 ISO9660 | 直接提取 |
| `CD001` @ 37649，大小能被 2352 整除 | 裸扇区 dump（MODE1/2352） | 先转 2048 再提取 |
| 两处都找不到 `CD001` | Akai 原生盘，无 ISO9660 | 直接提取，无需转换 |

第二种会在桌面生成 `<镜像名>-2048.iso` 中间文件。

---

## 注意事项

**会覆盖输出目录。** 每次运行前 `rm -rf` 掉 `<镜像名>_wav` 和 `<镜像名>_wav_stereo` 再重建。别把别的东西放进去。

**只扫桌面这一层。** 镜像放别处找不到。放桌面，或先 `mdfind -name xxx` 定位。

**脚本文件别加镜像后缀。** 扫描的是 `iso/img/cdr/bin`。把脚本改名成 `akai.bin` 放桌面，它会被当成镜像让你选编号。保持叫 `akai`。

**WAV 不带 loop 标记。** Akai 的 loop 点不会写进导出的 WAV。当素材拖进工程没问题；要在采样器里当乐器弹，loop 点得在 DAW 里重设。

**单声道不一定是问题。** 鼓库里 kick、snare、hat 这类单发音本来就是单声道，报告里出现属正常。

**镜像体积 ≠ 输出体积。** 505MB 镜像出 143MB WAV 很常见，Akai 盘常留空分区。只要采样数量级合理（几十到几百个）就是读全了。

---

## 故障排查

**提示"桌面没有找到 iso/img/cdr/bin 镜像"**
确认文件在桌面，后缀是这四种之一。

**提示"没有可用工具"**
手动跑 `python3 -m pip install --user smpl-extract`，或 `brew install libgig`。

**导出 0 个 WAV**
多半是混合模式盘——loop 存在 CD 音轨里，数据区只有单发音。这不是脚本的问题，需要 XLD / EAC 单独抓音轨。

**smpl-extract 报 `is not a file`**
这是 smpl-extract 外层解析器会重排未知选项，某些组合下把目标目录误当成镜像文件。脚本已内置三种参数写法依次尝试。手动跑遇到的话，改用：

```bash
cd 目标目录 && python3 -m smpl_extract export /完整路径/镜像.iso
```

**想确认盘有没有读全**
用 akaiextract 独立跑一遍交叉验证，件数应约等于 `单声道数 + 立体声数 × 2`。

---

## 许可证

脚本本体采用 **MIT License**，详见 [LICENSE](LICENSE)。

依赖的许可证各自独立，请自行遵守：

- [smpl-extract](https://github.com/DriftwoodAudio/smpl-extract) — 主提取器
- [libgig](http://www.linuxsampler.org/libgig/) — GPL，提供 `akaiextract`。本脚本仅调用其命令行程序、不链接其库

---

## 声明

**本仓库只包含脚本，不包含任何镜像文件或提取出的采样。**

采样盘的版权归原厂商（Best Service、Zero-G 等）所有，分发镜像属于侵权。本脚本仅用于帮助你处理自己合法持有的采样盘。
