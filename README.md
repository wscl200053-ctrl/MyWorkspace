# ✒️ Writer App - 跨平台沉浸式云同步写作软件

![Vue.js](https://img.shields.io/badge/Vue%203-35495E?style=for-the-badge&logo=vuedotjs&logoColor=4FC08D)
![TailwindCSS](https://img.shields.io/badge/Tailwind_CSS-38B2AC?style=for-the-badge&logo=tailwind-css&logoColor=white)
![Tauri](https://img.shields.io/badge/Tauri-FFC131?style=for-the-badge&logo=Tauri&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-007ACC?style=for-the-badge&logo=typescript&logoColor=white)

一款专为**长篇小说/网文作者**打造的跨平台写作软件。采用大前端技术栈开发，拥有极低的内存占用、优雅的沉浸式界面，以及**“离线优先、绝对防丢稿”**的本地存储机制。

本项目旨在实现：**一套代码，运行于 Windows、macOS 和 Android，并提供无缝的云端多端同步体验。**

## ✨ 当前核心特性 (Features)

- **📚 多书籍与树状目录**：支持在软件内创建多本书籍，并自由划分“卷”与“章”。
- **✍️ 沉浸式中文排版引擎**：基于 [Tiptap](https://tiptap.dev/) 深度定制。原生支持**中文首行缩进两字符**、优化行高与字间距，提供如纸张般纯净的输入体验。
- **💾 离线优先 & 极致防丢稿**：基于 `localForage` (IndexedDB) 构建底层数据库。无需网络，打字自动毫秒级防抖保存，彻底告别丢稿焦虑。断电重启，稿件依然如初。
- **⚡ 极速的桌面端底座**：借助 [Tauri](https://tauri.app/) 框架（底层为 Rust），彻底抛弃臃肿的 Electron，打包体积极小，启动秒开。
- **📊 实时字数统计**：右侧顶栏提供实时的纯文本字数统计与保存状态提示。

## 🛠️ 技术栈 (Tech Stack)

- **前端框架**: Vue 3 (Composition API) + TypeScript + Vite
- **UI 样式**: Tailwind CSS v4
- **富文本引擎**: Tiptap
- **本地数据库**: localForage (IndexedDB)
- **桌面端打包**: Tauri (Rust)
- **移动端打包**: Capacitor *(规划中)*
- **云同步后端**: Supabase (PostgreSQL) *(规划中)*

## 🚀 快速启动 (Getting Started)

确保你的电脑已安装 [Node.js](https://nodejs.org/) 和 [Rust](https://rustup.rs/) 环境。

```bash
# 1. 克隆本项目
git clone https://github.com/wscl200053-ctrl/MyWorkspace.git

# 2. 进入项目目录
cd writer-app

# 3. 安装依赖
npm install

# 4. 运行桌面开发环境 (首次运行需等待 Rust 编译)
npm run tauri dev

🤝 贡献与开源许可
本项目为个人全栈开发练习与自用工具，开源发布。欢迎提交 Issue 或 Pull Request 探讨交流！
