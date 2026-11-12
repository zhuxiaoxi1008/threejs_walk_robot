# 基于 Three.js 的 3D 角色控制与动画系统 - 技术总结

本项目基于原生 **Three.js** 构建，实现了一个带有原生动画、支持平滑移动、自动贴地及第三人称跟随的 3D 角色交互系统。

## 🎮 一、 项目特点 (Three.js 技术视角)

1. **原生动画状态机 (`AnimationMixer`)**：抛弃了早期手算正弦波的笨重方案，利用 `THREE.AnimationMixer` 提取模型自带的动画资产，通过 `fadeIn/fadeOut` 与 `LoopOnce` 实现了站立、奔跑与特技动作的无缝切换。
2. **包围盒自动贴地算法**：针对 3D 模型原点偏移的问题，利用 `THREE.Box3` 动态计算模型包围盒底部，在加载时自动将模型平移至网格地面（Y=0），确保脚底绝对贴合地面，无悬浮。
3. **平滑的第三人称控制**：结合 `THREE.MathUtils.lerp` 线性插值，实现了移动速度的平滑加速/减速、站立与行走动画的无缝切换，以及镜头的平滑跟随。

## 🚶 二、 运动与动画系统设计

### 1. 动画混合与过渡 (FadeIn/FadeOut)
* 封装 `fadeToAction(name, duration)` 方法。当切换动作时，让当前动作 `fadeOut`，目标动作 `fadeIn` 并 `play`，实现动作之间的平滑过渡，杜绝生硬闪切。
* 监听 `mixer.addEventListener('finished')` 事件：当“跳跃”、“挥手”等一次性动作播放完毕后，系统会自动判断当前是否有移动输入，从而平滑切回 `Idle` 或 `Walking`。

### 2. WASD 与 Shift 奔跑逻辑
* 监听 `keydown` 和 `keyup` 事件，维护一个按键状态字典 `keys = { w, a, s, d, shift }`。
* 根据按键状态计算目标速度。当按下 `Shift + W` 时，目标速度提升为 `runSpeed`，并自动将动画切换为 `Running`。

### 3. 三角位移算法
* 利用当前的朝向角度 `rotation.y`，通过 `Math.sin(rotation.y)` 和 `Math.cos(rotation.y)` 计算出 X 轴和 Z 轴的移动分量，让模型永远朝着它当前面朝的方向平移。

### 4. Delta Time 防滑步
* 使用 `const delta = clock.getDelta()` 记录两帧之间的时间差，位移和动画累加都必须乘以 `delta`，确保在不同刷新率（60Hz/144Hz）的显示器上，走路速度与步频始终一致。

## 🏗️ 三、 界面与场景架构 (Three.js 核心结构)

### 1. 基础渲染管线 (The Trinity)
* `THREE.Scene`：全局场景容器，配置浅绿背景色与 `Fog` 雾效。
* `THREE.PerspectiveCamera`：透视相机（视野 45°），呈现近大远小的真实空间感。
* `THREE.WebGLRenderer`：WebGL 渲染器，开启 `antialias`（抗锯齿）与 `shadowMap`（阴影），负责将场景转化为 Canvas 像素。

### 2. 资源加载与解析 (`GLTFLoader`)
* 采用 `GLTFLoader` 异步加载 `.glb` 模型资产，并通过遍历 `gltf.animations` 提取内置的所有动画片段存入字典。

### 3. 相机与交互系统 (`OrbitControls`)
* 引入 `OrbitControls` 作为基础的视角控制器，开启 `enableDamping`（阻尼）获得丝滑的鼠标拖拽手感。
* 使用 `controls.target.lerp(robotModel.position, 5 * delta)` 让相机的焦点实时、平滑地锁定在移动的模型身上，形成“跟拍”效果。

### 4. 环境与参照物辅助
* 使用 `GridHelper` 生成无限延伸的网格地面，作为 3D 空间的视觉参照系。
* 使用 `DirectionalLight`（平行光）产生锐利的阴影，结合 `AmbientLight`（环境光）照亮暗部，提升模型立体感。

## ⚡ 四、 优化动画性能 (结合 walkClip 实例)

在当前代码实现中，针对 `walkClip`（走路动画）以及其他动作的播放，我们做了以下几个核心的性能与体验优化：

1. **动作实例缓存 (`clipAction`)**：在模型加载完成后，统一使用 `mixer.clipAction(clip)` 将所有动画转为实例，并存入 `actions` 字典。避免了在 `animate` 循环或按键触发时重复创建实例，极大减少了内存开销和垃圾回收（GC）压力。
2. **底层 GPU 蒙皮计算**：通过 `AnimationMixer` 驱动动画，底层会自动使用 WebGL 的蒙皮网格（`SkinnedMesh`）着色器进行骨骼矩阵计算。将之前手写 `Math.sin` 的 CPU 计算压力转移到了 GPU，释放了主线程的性能。
3. **动态播放速率 (`timeScale`)**：结合移动速度，实时计算并更新 `actions['Walking'].timeScale`。当角色慢速移动时，降低动画播放帧率，避免了“滑步”现象，同时也减少了高频矩阵刷新带来的不必要的性能消耗。
4. **一次性动作自动截断 (`LoopOnce`)**：对于 `Jump`、`Punch` 等特技动作，强制设置 `setLoop(THREE.LoopOnce)`。动作播放一次后立刻停止并进入暂停状态，避免了无用动作在后台持续循环运算，由 `mixer` 的 `finished` 事件触发回调后，再按需唤醒 `Idle` 或 `Walking`。
5. **单次统一更新 (`mixer.update`)**：在 `animate` 的每一帧中，仅在最后调用一次 `mixer.update(delta)` 统一更新所有正在混合的动画轨道，既保证了动作的同步性，又避免了对动画系统进行重复的遍历更新。