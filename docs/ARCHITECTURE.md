<!-- 文档要点
- 聚焦仓库结构、模块职责与边界；接口、技术选型等可按需补充
- 只写跨模块的总览；单个模块的内部细节放到该模块目录的 README.md，在此留链接并登记到 AGENTS.md 文档索引
- 篇幅明显变长时（参考约 200 行），先检查是否混入了模块细节
-->
# ARCHITECTURE.md

## 仓库结构

- `index.html`：唯一交付物，单文件包含全部结构、样式与脚本
- `docs/PRODUCT.md`：产品需求与验收标准
- `docs/ARCHITECTURE.md`：本文档
- `AGENTS.md` / `CLAUDE.md` / `README.md` / `PROGRESS.md`：协作与进度文档

## 技术栈与引入

- Three.js **0.187.0**（锁定），通过动态 `import()` 双 CDN 源加载：主 npmmirror、备 jsdelivr，任一成功即用；双源均失败显示加载失败提示页（F6）
- 入口 `build/three.module.js` 相对导入 `three.core.js`，跟随源 URL 自动解析，无需 importmap
- 全部代码内联于单 HTML：`<style>` + `<script type="module">`，无构建步骤

## 模块职责与边界

单文件内按分节注释组织，各节职责：

| 模块 | 职责 | 边界 |
|---|---|---|
| 场景与灯光 | renderer / scene / camera 装配、月光与环境光、雾 | 不含具体物体建模 |
| 灯塔与礁石 | 塔身条纹、灯室、礁石群的几何建模 | 不含光束与动画 |
| 光束系统 | 半透明锥体 + 加色混合 + 灯室光晕、旋转驱动 | 只被动画 tick 驱动 |
| 海面 | PlaneGeometry + 顶点 shader 涌浪、片元着色 | 不依赖任何外部贴图 |
| 天空氛围 | 渐变天穹、星点、月亮、云 | 静态或自转，不响应交互 |
| 相机控制 | 手写轨道控制器（球坐标 + 阻尼）：拖拽旋转、缩放限位、俯仰限制、触屏 | 唯一处理输入的模块 |
| 加载兜底 | CDN / 模块加载失败的提示 UI | 独立于场景代码 |

边界规则：可调参数（波浪幅度、光束转速、颜色等）集中放在脚本顶部配置区；模块间不互相直接改状态，动画统一走一个 tick。

## 关键决策
<!-- 记录难以撤回或有真实取舍的决策。每条写：决策 / 原因 / 放弃的方案 -->

- **CDN 引入而非内嵌 three.js** / 保持单文件轻量、代码可读、版本可控 / 内嵌源码（离线可用但 700KB+ 难维护）
- **伪体积光束（锥体 + additive blending）而非 raymarching** / 视觉达标、性能好、实现与调试可控 / 真体积渲染（GPU 开销大、合成复杂、单文件内高风险）
- **手写 shader 海面而非官方 Water 示例** / 无外部贴图依赖，贴合单文件纯净化 / Water.js + normals 贴图（需 base64 内嵌使文件膨胀）
- **深夜场景而非黄昏或昼夜循环** / 体积光束对比度最高，星空月亮成本低 / 黄昏（光束存在感弱）、动态昼夜（光照系统复杂度陡增）
- **手写轨道控制器而非 OrbitControls addon** / 消除对 examples/jsm 的 CDN 依赖，使双源动态 import fallback 干净可行 / OrbitControls addon（多一层文件依赖，且部分源不含 addons 目录）
- **动态 import 双 CDN 源而非 importmap 单源** / T0 实测发现 CDN 单点不稳（jsdelivr HEAD 探测曾无响应），双源 try/catch 提升交付可靠性 / importmap（无法表达 fallback）
