# Sōzu 中文文档

本文档使用 AI 翻译

## 项目流程

### 1. 拉取上游文档
1. 创建空分支
```bash
git switch --orphan docs
```

2. 创建 `README.md`
```bash
cat > README.md <<EOF
# 中文文档

本文档使用 AI 翻译
EOF
```

3. 首次提交
```bash
git add .
git commit -am init
git push origin docs
```

4. 设置上游仓库
```bash
git remote add upstream https://github.com/sozu-proxy/sozu.git
git fetch upstream main
git checkout upstream/main -- doc
```

### 2. 安装 AI 助手
1. 安装 CLI 工具
```bash
npm install -g npm
npm install -g @google/gemini-cli
```

2. 设置环境变量
```bash
# 通过环境变量方式设置
export GEMINI_API_KEY=

# 通过 .env 文件配置
echo 'GEMINI_API_KEY=' > .env
```

3. AI 翻译
```bash
将 @doc 里面的英文文档翻译成中文，并且保存至 @docs_zh 文件夹里。
```

```bash
gemini --yolo --model "gemini-2.5-flash-lite" "将 @doc 里面的英文文档翻译成中文，并且保存至 @docs_zh 文件夹里。"
```

```bash
gemini --yolo --model "gemini-2.5-flash-lite" "推理过程使用中文输出。将 @doc 里面的英文文档翻译成中文，并且保存至 @docs_zh 文件夹里。"
```

## 文档管理器
- 安装 [Zensical](https://github.com/zensical/zensical)
```bash
pip install zensical
```