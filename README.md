# blog-source

Hexo + Butterfly 博客源码仓库，用于生成并部署 `timims.github.io`。

## 本地开发

```bash
npm install
npm run server
```

## 常用命令

```bash
npm run clean
npm run build
npm run deploy
```

## 仓库建议

- 页面仓库：`timims.github.io`
- 源码仓库：`blog-source`

## 评论系统

当前已经预留 Giscus 配置，但默认未启用。

启用步骤：

1. 在 GitHub 仓库开启 Discussions。
2. 到 `https://giscus.app/zh-CN` 生成配置。
3. 把生成的 `repo_id` 和 `category_id` 填入 [_config.butterfly.yml](/home/zkq/Projects/github/personal/blog-source/_config.butterfly.yml)。
4. 把 `comments.use` 设置为 `Giscus`。
