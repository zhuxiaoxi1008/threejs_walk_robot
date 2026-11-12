结合我们从零跑通模型、修复断脚、到实现 WASD 控制的全部过程，我为你梳理了纯 **Three.js 视角**的项目核心技术总结。

你可以直接拿这份总结去写技术博客或放入个人作品集。

---

### 🎮 一、 项目特点 (Three.js 技术视角)

1. **纯数学驱动的骨骼动画**：没有使用外部 3D 物理引擎（如 Cannon.js）或动画混合器（AnimationMixer），完全通过原生 `Math.sin` 正弦波函数驱动 `Bone` 骨骼节点旋转，实现“代码级”的步态控制。
2. **防御性代码与“骨骼外科手术”**：针对 Blender 导出的 `.glb` 模型普遍存在的“局部坐标系混乱”问题，利用 `Object3D.attach()` 及矩阵运算强行修复了断裂的脚部骨骼，并在动画循环中通过覆盖初始旋转值（`initRot`）来纠正倒挂的模型姿态。
3. **平滑的第三人称控制**：结合 `THREE.MathUtils.lerp` 线性插值，实现了移动速度的平滑加速/减速、站立与行走动画的无缝切换，以及镜头的平滑跟随。

---

### 🚶 二、 走路功能如何设计 (Three.js 骨骼与数学)

1. **正弦波与相位差 (Sine & Phase Shift)**
   * 使用 `Math.sin(time * speed)` 让大腿在 X 轴上做前后摆动。
   * 对右腿施加 `+ Math.PI`（180度）的相位差，保证左腿向前迈时，右腿刚好向后蹬，实现完美的双腿交替。
2. **单向折叠防骨折 (Clamping)**
   * 小腿（膝盖）的弯曲不能像大腿一样双向旋转，使用 `Math.max(0, -swing)` 截断负值，确保小腿永远只向后方折叠（或者向前，取决于坐标系），防止膝盖反向折断。
3. **WASD 与三角位移算法**
   * 监听 `keydown` 和 `keyup` 事件更新状态字典。
   * 在 `animate` 循环中，利用当前的朝向角度 `rotation.y`，通过 `Math.sin(rotation.y)` 和 `Math.cos(rotation.y)` 计算出 X 轴和 Z 轴的移动分量，让模型永远朝着它当前面朝的方向平移。
4. **Delta Time 防滑步**
   * 使用 `const delta = clock.getDelta()` 记录两帧之间的时间差，位移和动画累加都必须乘以 `delta`，确保在不同刷新率（60Hz/144Hz）的显示器上，走路速度与步频始终一致。

---

### 🏗️ 三、 界面与场景架构 (Three.js 核心结构)

1. **基础渲染管线 (The Trinity)**
   * `THREE.Scene`：承载一切的 3D 宇宙。
   * `THREE.PerspectiveCamera`：透视相机（视野 45°），呈现近大远小的真实空间感。
   * `THREE.WebGLRenderer`：WebGL 渲染器，开启 `antialias`（抗锯齿）与 `shadowMap`（阴影），负责将场景转化为 Canvas 像素。
2. **资源加载与解析 (GLTFLoader)**
   * 采用 `GLTFLoader` 异步加载 `.glb` 模型资产，并使用 `.traverse()` 遍历模型树（Scene Graph），通过 `getObjectByName()` 精准提取名为 `UpperLegL`、`FootL` 等骨骼节点。
3. **相机与交互系统 (OrbitControls)**
   * 引入 `OrbitControls` 作为基础的视角控制器，开启 `enableDamping`（阻尼）获得丝滑的鼠标拖拽手感。
   * 使用 `controls.target.lerp(robotModel.position, delta)` 让相机的焦点实时、平滑地锁定在移动的模型身上，形成“跟拍”效果。
4. **环境与参照物辅助**
   * 使用 `GridHelper` 生成无限延伸的网格地面，作为 3D 空间的视觉参照系。
   * 使用 `DirectionalLight`（平行光）产生锐利的阴影，结合 `AmbientLight`（环境光）照亮暗部，提升模型立体感。结合我们从零跑通模型、修复断脚、到实现 WASD 控制的全部过程，我为你梳理了纯 **Three.js 视角**的项目核心技术总结。

你可以直接拿这份总结去写技术博客或放入个人作品集。

---

### 🎮 一、 项目特点 (Three.js 技术视角)

1. **纯数学驱动的骨骼动画**：没有使用外部 3D 物理引擎（如 Cannon.js）或动画混合器（AnimationMixer），完全通过原生 `Math.sin` 正弦波函数驱动 `Bone` 骨骼节点旋转，实现“代码级”的步态控制。
2. **防御性代码与“骨骼外科手术”**：针对 Blender 导出的 `.glb` 模型普遍存在的“局部坐标系混乱”问题，利用 `Object3D.attach()` 及矩阵运算强行修复了断裂的脚部骨骼，并在动画循环中通过覆盖初始旋转值（`initRot`）来纠正倒挂的模型姿态。
3. **平滑的第三人称控制**：结合 `THREE.MathUtils.lerp` 线性插值，实现了移动速度的平滑加速/减速、站立与行走动画的无缝切换，以及镜头的平滑跟随。

---

### 🚶 二、 走路功能如何设计 (Three.js 骨骼与数学)

1. **正弦波与相位差 (Sine & Phase Shift)**
   * 使用 `Math.sin(time * speed)` 让大腿在 X 轴上做前后摆动。
   * 对右腿施加 `+ Math.PI`（180度）的相位差，保证左腿向前迈时，右腿刚好向后蹬，实现完美的双腿交替。
2. **单向折叠防骨折 (Clamping)**
   * 小腿（膝盖）的弯曲不能像大腿一样双向旋转，使用 `Math.max(0, -swing)` 截断负值，确保小腿永远只向后方折叠（或者向前，取决于坐标系），防止膝盖反向折断。
3. **WASD 与三角位移算法**
   * 监听 `keydown` 和 `keyup` 事件更新状态字典。
   * 在 `animate` 循环中，利用当前的朝向角度 `rotation.y`，通过 `Math.sin(rotation.y)` 和 `Math.cos(rotation.y)` 计算出 X 轴和 Z 轴的移动分量，让模型永远朝着它当前面朝的方向平移。
4. **Delta Time 防滑步**
   * 使用 `const delta = clock.getDelta()` 记录两帧之间的时间差，位移和动画累加都必须乘以 `delta`，确保在不同刷新率（60Hz/144Hz）的显示器上，走路速度与步频始终一致。

---

### 🏗️ 三、 界面与场景架构 (Three.js 核心结构)

1. **基础渲染管线 (The Trinity)**
   * `THREE.Scene`：承载一切的 3D 宇宙。
   * `THREE.PerspectiveCamera`：透视相机（视野 45°），呈现近大远小的真实空间感。
   * `THREE.WebGLRenderer`：WebGL 渲染器，开启 `antialias`（抗锯齿）与 `shadowMap`（阴影），负责将场景转化为 Canvas 像素。
2. **资源加载与解析 (GLTFLoader)**
   * 采用 `GLTFLoader` 异步加载 `.glb` 模型资产，并使用 `.traverse()` 遍历模型树（Scene Graph），通过 `getObjectByName()` 精准提取名为 `UpperLegL`、`FootL` 等骨骼节点。
3. **相机与交互系统 (OrbitControls)**
   * 引入 `OrbitControls` 作为基础的视角控制器，开启 `enableDamping`（阻尼）获得丝滑的鼠标拖拽手感。
   * 使用 `controls.target.lerp(robotModel.position, delta)` 让相机的焦点实时、平滑地锁定在移动的模型身上，形成“跟拍”效果。
4. **环境与参照物辅助**
   * 使用 `GridHelper` 生成无限延伸的网格地面，作为 3D 空间的视觉参照系。
   * 使用 `DirectionalLight`（平行光）产生锐利的阴影，结合 `AmbientLight`（环境光）照亮暗部，提升模型立体感。

---

### 优化动画性能
walkClip