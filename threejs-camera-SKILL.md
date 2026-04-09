---
name: threejs-camera
description: >
  Three.js 相機設定與控制的完整指引。當使用者在 Three.js 專案中涉及任何相機相關操作時，請務必使用此技能，包括：建立 PerspectiveCamera 或 OrthographicCamera、設定 fov / near / far / aspectRatio 參數、用滑鼠控制相機移動、整合 OrbitControls 或其他內建控制器、處理 Z-fighting 問題。即使使用者沒有明確說「相機」，只要涉及視角、視野、鏡頭、畫面比例、場景瀏覽互動，都應觸發此技能。
  本技能以 threejs-singleton-scene 的 Experience 架構為前提，所有相機操作都在 Camera.js 模組內進行。
---

# Three.js Camera Skill

本技能以 **Experience Singleton 架構**為前提。相機邏輯集中在 `src/Experience/Camera.js`，透過 `new Experience()` 取得 singleton 後存取所需資源。

---

## 架構中的 Camera 定位

```
Experience.js
  ├── this.sizes    ← Camera 需要取得 width / height / pixelRatio
  ├── this.scene    ← Camera instance 需要加入 scene
  ├── this.canvas   ← OrbitControls 需要綁定 canvas
  └── this.camera   ← Camera 類別實例（非 THREE.Camera）
        └── this.camera.instance  ← 實際的 THREE.PerspectiveCamera
```

`Experience` 統一往下呼叫 `camera.resize()` 與 `camera.update()`，Camera 本身不直接監聽 Sizes 的 resize event。

---

## 步驟一：判斷使用哪種相機

根據需求選擇相機類型，並告知使用者選擇原因：

- **PerspectiveCamera** — 模擬真實鏡頭透視感，**絕大多數場景使用這個**
- **OrthographicCamera** — 無透視，物體大小不隨距離變化，適合 RTS 遊戲、2D 等距視角
- `ArrayCamera` / `StereoCamera` / `CubeCamera` — 特殊用途，僅在使用者明確需要時提及

---

## 步驟二：建立 Camera.js

### PerspectiveCamera（預設）

```js
// src/Experience/Camera.js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import Experience from './Experience.js'

export default class Camera {
  constructor() {
    this.experience = new Experience()   // 取得 singleton，不傳任何參數
    this.sizes  = this.experience.sizes
    this.scene  = this.experience.scene
    this.canvas = this.experience.canvas

    this._setInstance()
    this._setControls()
  }

  _setInstance() {
    this.instance = new THREE.PerspectiveCamera(
      35,                                    // fov：建議 35~75
      this.sizes.width / this.sizes.height,  // aspectRatio：從 sizes 取得
      0.1,                                   // near
      100                                    // far
    )
    this.instance.position.set(6, 4, 8)
    this.scene.add(this.instance)
  }

  _setControls() {
    this.controls = new OrbitControls(this.instance, this.canvas)
    this.controls.enableDamping = true
  }

  // 由 Experience.resize() 呼叫，勿直接綁定 sizes.on('resize')
  resize() {
    this.instance.aspect = this.sizes.width / this.sizes.height
    this.instance.updateProjectionMatrix()
  }

  // 由 Experience.update() 呼叫
  update() {
    this.controls.update()
  }
}
```

> ⚠️ **context 陷阱**：不可寫 `this.sizes.on('resize', this.resize)`，callback 內的 `this` 會指向 `Sizes` 而非 `Camera`。請讓 `Experience` 統一呼叫 `camera.resize()`。

> ⚠️ **near / far 與 Z-fighting**：避免將 `near` 設極小（如 `0.0001`）、`far` 設極大（如 `9999999`），差距過大會導致兩個面互搶渲染順序（Z-fighting），畫面出現閃爍紋路。使用合理範圍（`0.1` / `100`），有需要再擴大。

---

### OrthographicCamera（特殊情境）

需要無透視感時，替換 `_setInstance()`：

```js
_setInstance() {
  const aspectRatio = this.sizes.width / this.sizes.height

  this.instance = new THREE.OrthographicCamera(
    -1 * aspectRatio,  // left
     1 * aspectRatio,  // right
     1,                // top
    -1,                // bottom
     0.1,              // near
     100               // far
  )
  this.instance.position.set(6, 4, 8)
  this.scene.add(this.instance)
}

// resize 也需更新 OrthographicCamera 的邊界
resize() {
  const aspectRatio = this.sizes.width / this.sizes.height
  this.instance.left   = -1 * aspectRatio
  this.instance.right  =  1 * aspectRatio
  this.instance.updateProjectionMatrix()
}
```

> ⚠️ `left / right` 必須乘以 `aspectRatio`，否則非正方形畫布會使物體變形拉伸。

---

## 步驟三：確認 Experience.js 正確呼叫

Camera 的 `resize()` 與 `update()` 由 Experience 統一觸發，確認 `Experience.js` 中已有：

```js
// Experience.js
resize() {
  this.camera.resize()
  this.renderer.resize()
}

update() {
  this.camera.update()   // controls.update() 在此執行
  this.world.update()
  this.renderer.update() // render call 永遠在最後
}
```

---

## 步驟四：OrbitControls 進階設定（選用）

在 `_setControls()` 內追加：

```js
_setControls() {
  this.controls = new OrbitControls(this.instance, this.canvas)
  this.controls.enableDamping = true

  // 選用設定
  this.controls.target.set(0, 1, 0)  // 改變觀看目標（預設場景中心）
  this.controls.rotateSpeed = 0.5
  this.controls.zoomSpeed = 1.2
  this.controls.minDistance = 2
  this.controls.maxDistance = 20
  this.controls.maxPolarAngle = Math.PI / 2  // 限制垂直旋轉角度
}
```

---

## 步驟五：自訂滑鼠控制（不使用 OrbitControls 時）

若需要完全自訂控制，在 Camera.js 內加入游標追蹤：

```js
constructor() {
  // ...
  this.cursor = { x: 0, y: 0 }
  this._setCursorListener()
  this._setInstance()
}

_setCursorListener() {
  this._onMouseMove = (event) => {
    this.cursor.x = event.clientX / this.sizes.width - 0.5
    this.cursor.y = -(event.clientY / this.sizes.height - 0.5)  // Y 軸反向
  }
  window.addEventListener('mousemove', this._onMouseMove)
}

// update() 中更新相機位置
update() {
  // 基本平移
  this.instance.position.x = this.cursor.x * 5
  this.instance.position.y = this.cursor.y * 5
  this.instance.lookAt(this.scene.position)

  // 或 360° 環繞
  this.instance.position.x = Math.sin(this.cursor.x * Math.PI * 2) * 5
  this.instance.position.z = Math.cos(this.cursor.x * Math.PI * 2) * 5
  this.instance.position.y = this.cursor.y * 3
  this.instance.lookAt(this.scene.position)
}

// destroy 時記得移除 listener
destroy() {
  window.removeEventListener('mousemove', this._onMouseMove)
}
```

> Y 軸取負號原因：Three.js Y 軸向上為正，瀏覽器 `clientY` 向下遞增，方向相反。

---

## 其他內建控制器（需要時才提及）

| 控制器 | 適用情境 |
|--------|----------|
| `TrackballControls` | 需要無限制垂直旋轉（可翻轉超過 180°） |
| `FlyControls` | 太空船式自由飛行 |
| `PointerLockControls` | FPS 遊戲（需自行實作位置與物理） |
| `TransformControls` / `DragControls` | 移動場景物體（與相機無關） |

所有控制器引入路徑格式相同：
```js
import { TrackballControls } from 'three/examples/jsm/controls/TrackballControls.js'
```

---

## 常見問題快速排查

| 症狀 | 原因 | 解法 |
|------|------|------|
| resize 後畫面比例錯誤 | 未更新 aspect 或未呼叫 updateProjectionMatrix | `resize()` 內確認兩行都有 |
| OrthographicCamera 物體變形 | `left/right` 未乘 aspectRatio | 乘上 `this.sizes.width / this.sizes.height` |
| 滑鼠 Y 軸方向相反 | 網頁 Y 軸與 Three.js Y 軸方向相反 | `cursor.y` 公式前加負號 |
| Z-fighting 閃爍 | near/far 差距過大 | 改用合理範圍如 `0.1` / `100` |
| OrbitControls damping 無效 | 未在每幀呼叫 update | 確認 `Experience.update()` 有呼叫 `camera.update()` |
| `OrbitControls is not defined` | 從 `THREE` 命名空間引入 | 改用 `three/examples/jsm/controls/OrbitControls.js` |
| destroy 後 controls 仍在運作 | 未呼叫 controls.dispose() | `Experience.destroy()` 中已有 `this.camera.controls.dispose()` |
