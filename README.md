# badge-print
吧唧打印图生成，用于上传裁剪图片并排版成可打印的尺寸

## 本地运行

```bash
pnpm install
pnpm dev
```

## 构建

```bash
pnpm build
```

构建结果会输出到 `dist` 目录。

## 部署

推荐用 Vercel 或 Netlify，项目已经包含对应配置。

### Vercel

1. 把项目上传到 GitHub。
2. 打开 https://vercel.com，新建项目并导入这个仓库。
3. Vercel 会读取 `vercel.json`，使用 `pnpm build` 构建，并发布 `dist`。

### Netlify

1. 把项目上传到 GitHub。
2. 打开 https://netlify.com，新建站点并导入这个仓库。
3. Netlify 会读取 `netlify.toml`，使用 `pnpm build` 构建，并发布 `dist`。
