---
layout: post
title: "发布npm包"
date: 2025-7-25
tags: [编程随笔]
comments: true
author: mazezen
---

## 发布 npm package

> 想要在包名字前面带 用户名或者组织名 如: npm install -g @abc/demo
> 需要购买 npm 平台 VIP.
> 发布就得用 npm publish --access public

1. 创建项目

2. 修改 package.json

```json
{
  "name": "demo",
  "version": "0.0.3",
  "main": "dist/index.js",
  "bin": {
    "demo": "dist/index.js"
  },
  "scripts": {
    "build": "tsc",
    "prepublishOnly": "npm run build"
  },
  "license": "MIT",
  "repository": {
    "type": "git",
    "url": ""
  }
}
```

3. 打包编译

```shell
npm run build
```

4. 修改 src/index.ts,添加

```js
#!/usr/bin/env node
```

5. 运行 npm login

```shell
npm login
```

6. 发布

- 手动发布

```shell
npm publish
```

- github 自动发布

* npm login 之后, 将生成的 token,添加到 github 仓库中
  `仓库 -> Settings -> Secrets and variables -> Actions -> New reposity secret`

* 在项目根目录下创建 `.github/workflows/publish.yml`

```yaml
name: Publish to npm
on:
  push:
    tags:
      - "v*"
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          registry-url: "https://registry.npmjs.org"
      - run: npm ci
      - run: npm run build
      - run: npm publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

- 终端执行

```shell
npm version patch
git push --follow-tags
```
