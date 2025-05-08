# Rust 编程教程

这是一个全面的 Rust 编程语言教程，涵盖从基础到高级的各个主题。

## 在线阅读

本教程已部署到 GitHub Pages，可以通过以下链接访问：

https://jyu213.github.io/rust-guide/

## 本地开发

### 环境准备

1. 安装 Python 和 pip
2. 安装依赖：
   ```bash
   pip install -r requirements.txt
   ```

### 本地预览

```bash
mkdocs serve
```

然后在浏览器中访问 http://localhost:8000

### 构建静态网站

```bash
mkdocs build
```

构建结果将生成在 `site` 目录中。

## 部署

本项目使用 GitHub Actions 自动部署到 GitHub Pages。当推送到 main 分支时，将自动触发构建和部署流程。

要手动部署，可以运行：

```bash
mkdocs gh-deploy
```

## 目录结构

- `docs/`: 所有 Markdown 文档
- `mkdocs.yml`: MkDocs 配置文件
- `.github/workflows/`: GitHub Actions 工作流配置

## 贡献指南

欢迎提交 Issue 或 Pull Request 来改进本教程。
