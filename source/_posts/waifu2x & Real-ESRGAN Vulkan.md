---
title: waifu2x & Real-ESRGAN Vulkan
date: 2026-11-17 00:00:00
updated: 2026-11-17 00:00:00 
description: Linux 环境下使用基于 Vulkan 的轻量级本地 AI 图像超分辨率（超分/放大）。
categories: 
- GNU/Linux
tags:
- waifu2x
- Real-ESRGAN
- 图片超分
---

Linux 环境下使用基于 Vulkan 的轻量级本地 AI 图像超分辨率（超分/放大）。

# 规范化部署与安装

软件实体统一归档至 `~/.local/opt`，通过包装脚本暴露到 `~/.local/bin`。

## 目录规划

- 安装根目录：`~/.local/opt/ai-upscale`
- 全局入口目录：`~/.local/bin`（需确保已加入系统 `PATH` 环境变量）

```sh
mkdir -p ~/.local/opt/ai-upscale ~/.local/bin
cd ~/.local/opt/ai-upscale
```

## 下载地址

### 官方直连地址

```
https://github.com/nihui/waifu2x-ncnn-vulkan/releases/download/20250915/waifu2x-ncnn-vulkan-20250915-linux.zip

https://github.com/xinntao/Real-ESRGAN/releases/download/v0.2.5.0/realesrgan-ncnn-vulkan-20220424-ubuntu.zip
```

### 镜像加速地址

```
https://ghfast.top/https://github.com/nihui/waifu2x-ncnn-vulkan/releases/download/20250915/waifu2x-ncnn-vulkan-20250915-linux.zip

https://ghfast.top/https://github.com/xinntao/Real-ESRGAN/releases/download/v0.2.5.0/realesrgan-ncnn-vulkan-20220424-ubuntu.zip
```

## 规范化解压

`bsdtar` 原生支持流式解析 `.zip` 格式，支持通过 `-C` 参数直接将内容解包至目标目录。

```bash
# waifu2x 自带顶层目录：解压后重命名为规范目录名
bsdtar -xf waifu2x-ncnn-vulkan-*-linux.zip
mv waifu2x-ncnn-vulkan-*-linux waifu2x

# Real-ESRGAN 压缩包内为平铺散文件：直接使用 -C 解包至目标子目录
mkdir -p realesrgan
bsdtar -xf realesrgan-ncnn-vulkan-*-ubuntu.zip -C realesrgan/

# 赋予主执行文件可执行权限，并清理 zip 压缩包
chmod +x waifu2x/waifu2x-ncnn-vulkan realesrgan/realesrgan-ncnn-vulkan
rm -f *.zip
```

# 核心参数详解与场景选型

## 通用核心参数解析

两个工具底层均基于 Tencent ncnn 推理框架，参数定义高度一致：

| 参数与说明 | 常见取值 | 作用与配置建议 |
| :--- | :--- | :--- |
| **`-n`**<br>`waifu2x: Denoise`（降噪等级） | `-1` / `0` / `1` / `2` / `3` | **降噪强度控制：**<br>• `-1`：纯放大不降噪（高清原图、矢量线稿推荐）<br>• `0`：极轻微去噪<br>• `1`：日常首选，轻度消除模糊杂讯，线条清晰锐利<br>• `2` ~ `3`：强力涂抹，专治高压缩、严重马赛克的 JPEG 旧图 |
| **`-n`**<br>`Real-ESRGAN: Model`（模型名称） | `realesr-animevideov3`<br>`realesrgan-x4plus` | **参数复用为指定模型：**<br>• `realesr-animevideov3`：二次元动漫首选，色彩不失真<br>• `realesrgan-x4plus`：真实摄影 / 3D 渲染，补充微观细节 |
| **`-s`**<br>两者通用：Scale（放大倍数） | 整数倍率 | **分辨率长宽缩放系数与模型约束：**<br>• `waifu2x` 支持 `1` / `2` / `4` / `8` / `16` / `32`（默认 `2`）<br>• `realesr-animevideov3` 仅支持 **`2` / `3` / `4`** 倍（默认 `4`）<br>• `realesrgan-x4plus` 仅支持 **`4`** 倍 |
| **`-f`**<br>两者通用：Format（输出格式） | `png` / `jpg` / `webp` | **图像编码容器：**<br>• `png`（首选）：无损保存，防止放大后二次压缩劣化<br>• `webp`：体积更小，兼顾质量，适合超大壁纸长期存储 |
| **`-g`**<br>两者通用：GPU ID（设备索引） | `0` / `1` / `...` | **运行设备指定：**<br>双显卡机器必须显式指定独显（通常为 `1`，可在日志确认）；误走核显会导致处理速度降低数倍 |
| **`-t`**<br>两者通用：Tile Size（切片大小） | `0` / `100` / `200` / `400` | **显存切片分块粒度：**<br>• 默认 `0` 自动计算；显存充裕建议设为 `200` 或 `400`<br>• 处理超大图显存溢出（Vulkan OOM）时，缩减至 `100` |
| **`-m`**<br>两者通用：Model Path（权重路径） | 目录绝对路径 | **模型文件查找目录：**<br>二进制默认按当前工作目录寻址，务必传绝对路径（如 `~/.local/opt/.../models`），避免切换目录执行时报错闪退 |

## 场景与模型选型矩阵

| 场景 | 推荐引擎 | 推荐模型/核心参数 | 核心特性 |
| :--- | :--- | :--- | :--- |
| **二次元插画 / 赛璐珞动漫** | `waifu2x` | `-m models-cunet -s 2 -n 1` | 线条锐利干净，不生造杂质纹理，对 2D 平面效果极佳。 |
| **动漫重度压缩噪点修复** | `waifu2x` | `-m models-cunet -n 2` | 强力消除网络图多次转压产生的振铃效应与 JPEG 杂色块。 |
| **写实照片 / 3D 渲染** | `Real-ESRGAN` | `-n realesrgan-x4plus -s 4` | 针对现实纹理（发丝、布料、皮肤肌理）自动补充微观细节。 |
| **复杂场景动漫壁纸 4X 放大** | `Real-ESRGAN` | `-n realesr-animevideov3 -s 4` | 专为动漫微调的轻量模型，色彩保持度高且线条不易产生畸变。 |

# CLI 快速调用示例

## 案例 A：waifu2x

二次元插画 2 倍放大 + 1 级降噪 + PNG 格式输出：

```sh
~/.local/opt/ai-upscale/waifu2x/waifu2x-ncnn-vulkan \
  -i input.png \       # 输入图像路径（支持 jpg/png/webp 或整目录）
  -o output_2x.png \   # 输出图像路径（支持 jpg/png/webp 或整目录）
  -n 1 \               # 降噪等级（-1/0/1/2/3，默认 0；1 级兼顾降噪与线条）
  -s 2 \               # 放大倍数（1/2/4/8/16/32，默认 2）
  -f png \             # 输出图像编码格式（jpg/png/webp，默认随扩展名/png）
  -t 200 \             # 显存切片分块尺寸（>=32/0=自动，大图防爆显存填 100/200）
  -g 1 \               # 指定运行的 GPU 设备（-1 为 CPU，0 为核显，1 为独显）
  -m ~/.local/opt/ai-upscale/waifu2x/models-cunet # 模型权重绝对路径（默认为 models-cunet）
```

## 案例 B：Real-ESRGAN

动漫模型 4 倍放大 + PNG 格式输出：

```sh
~/.local/opt/ai-upscale/realesrgan/realesrgan-ncnn-vulkan \
  -i input.png \             # 输入图像路径（支持 jpg/png/webp 或整目录）
  -o output_4x.png \         # 输出图像路径（支持 jpg/png/webp 或整目录）
  -n realesr-animevideov3 \  # 预训练模型名称（动漫推荐使用 realesr-animevideov3）
  -s 4 \                     # 放大倍数（支持 2/3/4，默认 4）
  -f png \                   # 输出图像编码格式（jpg/png/webp，默认随扩展名/png）
  -t 200 \                   # 显存切片分块尺寸（>=32/0=自动，大图防爆显存填 100/200）
  -g 1 \                     # 指定运行的 GPU 设备（-1 为 CPU，0 为核显，1 为独显）
  -m ~/.local/opt/ai-upscale/realesrgan/models # 模型权重目录绝对路径（默认为 models）
```

# 封装一键全局命令

在 `~/.local/bin/upscale` 创建包装脚本，支持直接传参模式与零参数交互式向导模式，实现全系统任意目录一键调用。

## 存放路径

```
~/.local/bin/upscale
```

_(文件写入后需执行 `chmod +x ~/.local/bin/upscale` 赋予可执行权限)_

## 脚本源码 (`upscale`)

```sh
#!/usr/bin/env bash
# 严谨模式：未定义变量报错、管道故障退出
set -euo pipefail

# ==================== 基础路径与默认硬件配置 ====================
BASE_OPT="$HOME/.local/opt/ai-upscale"
DEFAULT_GPU=1  # 默认使用独显 (0 为核显，1 为独显)

# 打印帮助信息
usage() {
    cat << HELP
用法:
  交互模式: 直接执行 upscale
  命令模式: upscale [选项] -i <输入图片> -o <输出图片>

选项:
  -m, --model     模型引擎 (默认: waifu2x)
                    - waifu2x               : 2D动漫/插画 (保持线条锐利，不生杂质纹理)
                    - realesr-animevideov3  : 复杂动漫场景超分 (色彩好，画质硬朗)
                    - realesrgan-x4plus     : 照片/3D材质 (增强微观发丝/织物纹理)
  -n, --noise     waifu2x 降噪等级: -1/0/1/2/3 (默认 1，仅 waifu2x 生效)
  -s, --scale     放大倍数: 整数 (waifu2x 默认 2，Real-ESRGAN 系列默认 4)
  -f, --format    输出格式: png | jpg | webp (默认 png)
  -t, --tile      切片尺寸: 0=自动，超大图防爆显存填 100/200 (默认 0)
  -x, --tta       启用 TTA 极限画质模式 (仅 waifu2x 支持，耗时提升8倍)
  -g, --gpu       GPU 设备 ID (默认 1)
  -h, --help      查看帮助
HELP
    exit 0
}

# ==================== 变量初始化 ====================
MODEL=""
NOISE=""
SCALE=""
FORMAT=""
TILE=""
GPU_ID=""
TTA=0
INPUT=""
OUTPUT=""

# ==================== CLI 参数解析 ====================
while [[ $# -gt 0 ]]; do
    case "$1" in
        -m|--model)  MODEL="$2"; shift 2 ;;
        -n|--noise)  NOISE="$2"; shift 2 ;;
        -s|--scale)  SCALE="$2"; shift 2 ;;
        -f|--format) FORMAT="$2"; shift 2 ;;
        -t|--tile)   TILE="$2"; shift 2 ;;
        -g|--gpu)    GPU_ID="$2"; shift 2 ;;
        -x|--tta)    TTA=1; shift ;;
        -i)          INPUT="$2"; shift 2 ;;
        -o)          OUTPUT="$2"; shift 2 ;;
        -h|--help)   usage ;;
        *)           echo "错误: 未知参数 $1" >&2; usage ;;
    esac
done

# ==================== 交互向导模式 (无参数时触发) ====================
if [[ -z "$INPUT" && -z "$OUTPUT" ]]; then
    echo "=================================================="
    echo "        AI 图像超分辨率交互运行向导 (Vulkan)       "
    echo "=================================================="

    # 1. 输入文件
    while true; do
        read -rp "1. 输入图片路径 (拖拽文件进终端或输入路径): " INPUT
        # 清理路径外层可能被终端自动包裹的引号与空格
        INPUT=$(echo "$INPUT" | sed -e "s/^['\"]//" -e "s/['\"]$//" | xargs)
        if [[ -f "$INPUT" ]]; then
            break
        else
            echo "   [错误] 文件不存在，请重新输入！"
        fi
    done

    # 2. 模型选型
    echo ""
    echo "2. 选择模型引擎:"
    echo "   [1] waifu2x              - 2D动漫/插画首选 (线条干净、可降噪)"
    echo "   [2] realesr-animevideov3 - 动漫壁纸 4X 超分 (细节硬朗、色彩自然)"
    echo "   [3] realesrgan-x4plus    - 真实照片/3D渲染 (补充微观材质细节)"
    read -rp "   请选择 [1-3] (默认: 1): " MODEL_CHOICE
    case "${MODEL_CHOICE:-1}" in
        1) MODEL="waifu2x" ;;
        2) MODEL="realesr-animevideov3" ;;
        3) MODEL="realesrgan-x4plus" ;;
        *) echo "   输入有误，自动采用默认值: waifu2x"; MODEL="waifu2x" ;;
    esac

    # 3. 放大倍数与降噪配置
    echo ""
    if [[ "$MODEL" == "waifu2x" ]]; then
        read -rp "3. 放大倍数 [1, 2, 4, 8, 16, 32] (默认: 2): " SCALE
        SCALE="${SCALE:-2}"

        echo ""
        echo "4. waifu2x 降噪等级 (-n):"
        echo "   [-1] 纯放大不降噪 (高清无损原图、线稿推荐)"
        echo "   [ 0] 极轻微去噪"
        echo "   [ 1] 日常推荐 (消除噪点且保全线条)"
        echo "   [ 2-3] 强力涂抹 (重度网图/JPEG马赛克)"
        read -rp "   请设置 [-1 到 3] (默认: 1): " NOISE
        NOISE="${NOISE:-1}"

        read -rp "5. 开启 TTA 8倍画质模式? 极慢但线条更纯净 [y/N] (默认: N): " TTA_CONFIRM
        if [[ "${TTA_CONFIRM:-N}" =~ ^[Yy]$ ]]; then
            TTA=1
        fi
    else
        if [[ "$MODEL" == "realesr-animevideov3" ]]; then
            ALLOWED_SCALES="2 3 4"
            SCALE_PROMPT="3. 放大倍数 [2, 3, 4] (默认: 4): "
            SCALE_ERROR="realesr-animevideov3 仅支持 2/3/4 倍，没有 1x 模型"
        else
            ALLOWED_SCALES="4"
            SCALE_PROMPT="3. 放大倍数 [4] (默认: 4): "
            SCALE_ERROR="realesrgan-x4plus 仅支持 4 倍"
        fi

        while true; do
            read -rp "$SCALE_PROMPT" SCALE
            SCALE="${SCALE:-4}"
            if [[ " $ALLOWED_SCALES " == *" $SCALE "* ]]; then
                break
            fi
            echo "   [错误] $SCALE_ERROR，请重新输入。"
        done
    fi

    # 4. 输出格式与路径命名推导
    echo ""
    read -rp "6. 输出格式 [png/jpg/webp] (默认: png): " FORMAT
    FORMAT="${FORMAT:-png}"

    # 提取无后缀的原图路径前缀（如 /path/to/image）
    BASE_NAME="${INPUT%.*}"

    # 根据工具与模型组装默认命名规则
    if [[ "$MODEL" == "waifu2x" ]]; then
        # 格式：原图名称_waifu2x_倍率x.格式
        DEFAULT_OUT="${BASE_NAME}_waifu2x_${SCALE}x.${FORMAT}"
    else
        # 格式：原图名称_realesrgan_模型名_倍率x.格式
        DEFAULT_OUT="${BASE_NAME}_realesrgan_${MODEL}_${SCALE}x.${FORMAT}"
    fi

    echo ""
    read -rp "7. 输出路径 (直接回车默认: $DEFAULT_OUT): " OUTPUT
    OUTPUT="${OUTPUT:-$DEFAULT_OUT}"

    # 5. 高级性能项
    echo ""
    read -rp "8. 显存切片 Tile 大小 (默认 0 自动计算，大图防爆显存填 100/200): " TILE
    TILE="${TILE:-0}"

    read -rp "9. GPU 设备 ID (0=核显，1=独显，默认: $DEFAULT_GPU): " GPU_ID
    GPU_ID="${GPU_ID:-$DEFAULT_GPU}"

    echo "=================================================="
    echo ">> 任务配置完成，开始推理..."
fi

# ==================== 默认值保底填充 (针对 CLI 传参模式) ====================
MODEL="${MODEL:-waifu2x}"
NOISE="${NOISE:-1}"
TILE="${TILE:-0}"
FORMAT="${FORMAT:-png}"
GPU_ID="${GPU_ID:-$DEFAULT_GPU}"

if [[ -z "$SCALE" ]]; then
    if [[ "$MODEL" == "waifu2x" ]]; then
        SCALE=2
    else
        SCALE=4
    fi
fi

# 校验基础文件
if [[ -z "$INPUT" || -z "$OUTPUT" ]]; then
    echo "错误: 必须指定 -i 与 -o" >&2
    usage
fi

if [[ ! -f "$INPUT" ]]; then
    echo "错误: 输入文件不存在 '$INPUT'" >&2
    exit 1
fi

# ==================== 组装参数并执行引擎 ====================
EXTRA_ARGS=()
[[ "$TILE" != "0" ]] && EXTRA_ARGS+=(-t "$TILE")

case "$MODEL" in
    waifu2x)
        [[ "$TTA" -eq 1 ]] && EXTRA_ARGS+=(-x)
        exec "$BASE_OPT/waifu2x/waifu2x-ncnn-vulkan" \
            -i "$INPUT" \
            -o "$OUTPUT" \
            -n "$NOISE" \
            -s "$SCALE" \
            -f "$FORMAT" \
            -g "$GPU_ID" \
            -m "$BASE_OPT/waifu2x/models-cunet" \
            "${EXTRA_ARGS[@]}"
        ;;
    realesr-animevideov3)
        exec "$BASE_OPT/realesrgan/realesrgan-ncnn-vulkan" \
            -i "$INPUT" \
            -o "$OUTPUT" \
            -n realesr-animevideov3 \
            -s "$SCALE" \
            -f "$FORMAT" \
            -g "$GPU_ID" \
            -m "$BASE_OPT/realesrgan/models" \
            "${EXTRA_ARGS[@]}"
        ;;
    realesrgan-x4plus)
        exec "$BASE_OPT/realesrgan/realesrgan-ncnn-vulkan" \
            -i "$INPUT" \
            -o "$OUTPUT" \
            -n realesrgan-x4plus \
            -s "$SCALE" \
            -f "$FORMAT" \
            -g "$GPU_ID" \
            -m "$BASE_OPT/realesrgan/models" \
            "${EXTRA_ARGS[@]}"
        ;;
    *)
        echo "错误: 未知模型 '$MODEL'，支持的模型: waifu2x | realesr-animevideov3 | realesrgan-x4plus" >&2
        exit 1
        ;;
esac
```

## 日常调用方式

### 交互向导模式

终端直接敲命令，跟随引导回车配置：

```sh
upscale
```

### 命令行直接调用

```sh
# waifu2x 二次元插画日常放大（2倍 + 1级降噪）
upscale -m waifu2x -i wallpaper.png -o wallpaper_2x.png

# 复杂二次元大图 4 倍超分，并输出为无损 webp
upscale -m realesr-animevideov3 -s 4 -f webp -i input.jpg -o output_4x.webp

# 真实拍摄照片 / 3D 渲染 4 倍细节超分
upscale -m realesrgan-x4plus -i portrait.jpg -o portrait_4x.png
```

# 最终目录结构

部署完成后，家目录中的文件树布局如下：

```
~/.local/
├── bin/
│   └── upscale                        # 一键封装脚本（在 PATH 中，全局可直接调用）
└── opt/
    └── ai-upscale/
        ├── realesrgan/                # Real-ESRGAN 独立闭包
        │   ├── models/                # 模型权重文件 (*.bin, *.param)
        │   └── realesrgan-ncnn-vulkan # 主可执行文件
        └── waifu2x/                   # waifu2x 独立闭包
            ├── models-cunet/          # CUNet 模型权重
            ├── models-upconv_7_anime_style_art_rgb/
            ├── models-upconv_7_photo/
            └── waifu2x-ncnn-vulkan    # 主可执行文件
```

# 卸载

由于遵循了 `~/.local/opt` 的应用隔离规范，没有向系统其他路径散落杂乱文件，卸载只需删除对应的目录和执行脚本即可：

```sh
# 1. 移除模型与引擎实体文件
rm -rf ~/.local/opt/ai-upscale

# 2. 移除全局调用命令
rm -f ~/.local/bin/upscale
```

# PNG 无损压缩

如果前面放大选择 PNG，得到的图片体积一般都比较大，可以使用 oxipng 进行无损压缩：

```bash
oxipng -o4 --strip safe <path/to/picture>
```

而选择 JPG 因为编码格式原因（有损 q=100，4:4:4，误差存在但极小），体积一般会比 PNG 小不少，并且无需进行手动压缩。
