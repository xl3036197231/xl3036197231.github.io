# 小星的数字花园

一个使用 Hugo 和 PaperMod 构建的中文个人博客。

## 本地预览

```bash
hugo server --bind 0.0.0.0 --port 1313 --disableFastRender --renderToMemory --environment development
```

打开 `http://localhost:1313/` 即可预览。开发环境会自动使用本地地址，不影响生产配置。

## 生产构建

```bash
hugo --cleanDestinationDir --gc --minify --panicOnWarning
```

生成文件位于 `public/`，该目录由构建产生，不提交到 Git。

## 发布

推送至 `main` 会触发 `.github/workflows/deploy-pages.yml`，自动构建并部署到 GitHub Pages。
