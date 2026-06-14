# Cyber-Snake

赛博风贪吃蛇纯前端版，不依赖 Python 后端或 MySQL。游戏、分数和最高分都在浏览器本地运行，适合部署到 GitHub Pages、Vercel、Netlify，也可以直接打包成静态文件发给别人。

## Features

- Vue 3 + Vite + TypeScript
- Canvas 绘制赛博风蛇身、蛇头和食物
- 支持方向键和 WASD
- 本地最高分保存在 `localStorage`
- 无后端、无数据库、无在线排行榜

## Development

```powershell
npm install
npm run dev
```

## Build

```powershell
npm run build
```

构建产物在 `dist/`。
