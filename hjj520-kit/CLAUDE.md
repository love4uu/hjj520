# 表白网站工坊 — AI 操作手册

## 这是什么

一个浪漫表白网站的模板系统。客户提供照片/视频/文字，你生成一个粉色系、带背景音乐和动画的单页网站，部署到 GitHub Pages。

成品效果：https://love4uu.github.io/love/Hina520/

## 文件夹结构

```
hjj520-kit/
├── CLAUDE.md          ← 你正在读的文件
├── github_token.txt   ← GitHub Personal Access Token（用户提供）
├── template/
│   └── index.html     ← 网站模板，含 {{占位符}}
├── shared/
│   └── bgm.mp3        ← 默认背景音乐
└── orders/            ← 订单放这里
    └── 客户名/
        ├── order.txt  ← 客户信息
        ├── photo*.jpg ← 客户照片（原图）
        └── video.mp4  ← 客户视频（原文件）
```

## 一次性环境准备

用户换新电脑后，你检查并安装依赖：

### 1. 安装 FFmpeg
```bash
winget install --id Gyan.FFmpeg --accept-source-agreements --accept-package-agreements
```

### 2. 安装 GitHub CLI
```bash
winget install --id GitHub.cli --accept-source-agreements --accept-package-agreements
```

### 3. 配置 GitHub 认证

检查 `github_token.txt` 是否存在。如果没有，告诉用户：
"请去 github.com → Settings → Developer settings → Personal access tokens → Generate new token (classic)，勾选 repo 权限，把 token 贴给我。"

拿到 token 后配置：
```bash
git config --global user.name "love4uu"
git config --global user.email "用户邮箱"
git config --global credential.helper store
# 然后用 token 作为密码进行 git 操作
```

### 4. 克隆接单仓库
```bash
git clone https://github.com/love4uu/love.git "Z:/workspace/love"
```
如已存在则 `git pull`。

## 接单流程

用户把订单素材放到 `orders/客户名/` 下，附带 `order.txt`。

### order.txt 格式

```
名字=Hina
标题=5201314
情书=我是森岛帆高
后缀=Hina520
照片数=6
BGM=默认
视频=video.mp4
```

### 处理步骤

#### 1. 读取 order.txt，确认信息

#### 2. 创建客户目录
```bash
mkdir -p "Z:/workspace/love/客户后缀"
```

#### 3. 处理照片

复制原图并重命名为 photo1.jpg ~ photoN.jpg：
```bash
cp "orders/客户名/原始照片1.jpg" "Z:/workspace/love/客户后缀/photo1.jpg"
# ... 逐张复制
```

生成 WebP 缩略图（400px 宽）：
```bash
FFMPEG="/path/to/ffmpeg.exe"
for i in $(seq 1 照片数); do
  "$FFMPEG" -i "Z:/workspace/love/客户后缀/photo${i}.jpg" \
    -vf "scale=400:-1" -q:v 75 \
    "Z:/workspace/love/客户后缀/photo${i}_thumb.webp" -y
done
```

#### 4. 处理视频

压缩视频（720p H.264，CRF 26）：
```bash
"$FFMPEG" -i "orders/客户名/视频文件.mp4" \
  -vf "scale=720:-2" -c:v libx264 -crf 26 -preset medium \
  -c:a aac -b:a 96k -movflags +faststart \
  "Z:/workspace/love/客户后缀/video.mp4" -y
```

截取首帧做封面：
```bash
"$FFMPEG" -i "Z:/workspace/love/客户后缀/video.mp4" \
  -ss 00:00:01 -vframes 1 -q:v 75 \
  "Z:/workspace/love/客户后缀/poster.webp" -y
```

#### 5. 处理 BGM

如果 order.txt 写 `BGM=默认`：
```bash
BGM_PATH="../shared/bgm.mp3"
```
如果客户自带 BGM，复制到客户目录并用 `bgm.mp3`。

#### 6. 生成 index.html

从 `template/index.html` 复制，替换占位符：

| 占位符 | 替换为 |
|--------|--------|
| `{{NAME}}` | 名字 |
| `{{TITLE}}` | 标题 |
| `{{LETTER}}` | 情书内容 |
| `{{REPO}}` | `love` |
| `{{FOLDER}}` | 客户后缀 |
| `{{BGM_SRC}}` | `../shared/bgm.mp3` 或 `bgm.mp3` |
| `<!-- {{PHOTOS}} -->` | 生成 N 张 img 标签（`photo1_thumb.webp` ~ `photoN_thumb.webp`，4:3 比例，lazy loading） |

替换命令：
```bash
cp template/index.html "Z:/workspace/love/客户后缀/index.html"
sed -i \
  -e 's|{{NAME}}|名字|g' \
  -e 's|{{TITLE}}|标题|g' \
  -e 's|{{LETTER}}|情书|g' \
  -e 's|{{REPO}}|love|g' \
  -e 's|{{FOLDER}}|客户后缀|g' \
  -e 's|{{BGM_SRC}}|BGM路径|g' \
  "Z:/workspace/love/客户后缀/index.html"
```

照片标签用 awk 替换 `<!-- {{PHOTOS}} -->`：
```bash
awk -v photos="生成的N张img标签\n" \
  '{ gsub(/<!-- \{\{PHOTOS\}\} -->/, photos); print }' \
  index.html > index_new.html && mv index_new.html index.html
```

每张照片标签格式：
```html
<img src="photo1_thumb.webp" alt="合照1" loading="lazy" width="200" height="150" style="width:100%;aspect-ratio:4/3;object-fit:cover;border-radius:12px;">
```

#### 7. 部署

```bash
cd "Z:/workspace/love"
git add "客户后缀/"
git commit -m "新订单: 客户后缀 - 照片数张照片"
git push origin main
```

#### 8. 告诉用户

> 已部署：https://love4uu.github.io/love/客户后缀/

## FFmpeg 路径

FFmpeg 通过 winget 安装后在：
```
C:\Users\[用户名]\AppData\Local\Microsoft\WinGet\Packages\Gyan.FFmpeg_Microsoft.Winget.Source_8wekyb3d8bbwe\ffmpeg-*-full_build\bin\ffmpeg.exe
```
用 `find` 命令自动定位：
```bash
FFMPEG=$(find /c/Users -name "ffmpeg.exe" -path "*/Gyan.FFmpeg*" 2>/dev/null | head -1)
```

## 重要规则

- 照片比例默认 4:3，aspect-ratio CSS 控制
- 视频压缩到 720p，目标 3-8MB
- 原图保留在客户目录作为备份（不用于页面加载）
- 每次接单后在订单目录下记录 order.txt
- git push 后 1-2 分钟 GitHub Pages 生效
