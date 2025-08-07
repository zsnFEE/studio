# 音频编辑器界面

基于 Vue3 + T-Design 实现的音频编辑器界面，复原了原始设计图的所有功能和视觉效果。

## 功能特点

- 🎵 **多轨道音频编辑** - 支持多个音频轨道的同时编辑
- 🎛️ **实时波形显示** - 动态生成的音频波形可视化
- ⏯️ **播放控制** - 完整的播放/暂停/拖拽控制
- 🏷️ **音频片段管理** - 音频片段的展示、选择和标签系统
- 🎨 **现代化UI** - 基于 T-Design 的美观界面
- ✨ **动画效果** - 装饰性动画和交互反馈

## 技术栈

- **Vue 3** - 渐进式 JavaScript 框架
- **T-Design Vue Next** - 腾讯企业级设计语言的 Vue 3 实现
- **Vite** - 下一代前端构建工具

## 项目结构

```
src/
├── components/
│   ├── AudioClipItem.vue      # 音频片段组件
│   ├── TimelineControls.vue   # 时间轴控制组件
│   ├── WaveformTracks.vue     # 多轨道容器组件
│   └── WaveformTrack.vue      # 单个轨道组件
├── App.vue                    # 主应用组件
└── main.js                    # 应用入口
```

## 安装和运行

1. 安装依赖：
```bash
npm install
```

2. 启动开发服务器：
```bash
npm run dev
```

3. 构建生产版本：
```bash
npm run build
```

## 主要组件说明

### AudioClipItem.vue
- 显示音频片段的缩略图、标题、时长
- 支持标签显示和选择状态
- 动态生成 SVG 缩略图

### TimelineControls.vue
- 播放/暂停控制
- 时间显示和进度条
- 可拖拽的播放头
- 时间刻度标记

### WaveformTracks.vue & WaveformTrack.vue
- 多轨道音频波形显示
- Solo/Mute 控制按钮
- 动态生成的波形可视化
- 轨道选择和高亮效果

## 特色功能

1. **动态波形生成** - 使用数学函数生成逼真的音频波形
2. **装饰动画** - 右侧的彩色圆点浮动动画
3. **响应式设计** - 适配不同屏幕尺寸
4. **T-Design 组件** - 使用企业级 UI 组件库
5. **模块化架构** - 组件间清晰的数据流和事件通信

## 浏览器支持

- Chrome >= 80
- Firefox >= 75
- Safari >= 13
- Edge >= 80