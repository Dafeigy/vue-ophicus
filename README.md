# PROJECT OPHICULUS

基于 Vue 3 + Vite 的项目，实现了摄像头调用和二维码扫描功能。

## 功能特性

- 实时摄像头调用
- 持续二维码扫描并打印结果
- 响应式设计，支持移动端和桌面端
- Element Plus UI 组件

## 技术栈

- Vue 3
- Vite
- Element Plus
- jsQR (二维码扫描)
- Tailwind CSS

## 开发说明

### 安装依赖

```bash
npm install
```

### 开发环境运行

```bash
npm run dev
```

### 构建生产版本

```bash
npm run build
```

## Vercel 部署说明

本项目已配置好 Vercel 部署所需的文件：

- `vercel.json` - Vercel 配置文件，指定了构建命令和输出目录
- 构建输出目录: `dist`
- 构建命令: `npm run build`

### 部署步骤

1. 将代码推送到 GitHub/GitLab 仓库
2. 登录 Vercel 账号
3. 点击 "New Project"
4. 导入你的仓库
5. Vercel 会自动检测到 `vercel.json` 配置
6. 点击 "Deploy" 开始部署

## 注意事项

- 摄像头功能需要 HTTPS 环境或 localhost 访问
- 在移动端可能需要额外的摄像头权限配置
