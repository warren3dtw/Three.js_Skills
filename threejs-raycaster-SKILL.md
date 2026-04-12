---
name: threejs-raycaster
description: >
  當使用者在 Three.js 專案中涉及任何光線投射、滑鼠互動、物件點擊、hover 效果、
  滑鼠懸停偵測、3D 場景中的滑鼠事件時，請使用此 skill。
  觸發情境包括：偵測滑鼠是否移到某個物件上、點擊 3D 物件觸發事件、
  判斷光線是否與物件相交、在動畫中偵測物體碰撞、GLTF 模型的點擊偵測。
  即使使用者沒有說「Raycaster」，只要涉及「點擊物件」、「滑鼠移到上面」、
  「hover 效果」、「3D 點擊」，都應觸發此 skill。
  本技能以 threejs-experience 的 Experience 架構為前提。
---

# Three.js Raycaster Skill

本技能以 **Experience Singleton 架構**（`threejs-experience`）為前提。Raycaster 邏輯集中在 `src/Experience/Raycaster.js`，繼承 `EventEmitter`，透過 `on` / `trigger` 與 World 子模組溝通，與整個 Experience 架構的事件系統一致。

---

## Raycaster 的核心概念

Raycaster 從一個起點向特定方向射出一條射線，並回傳所有與該射線相交的物件。

主要應用：
- 滑鼠 hover 效果（哪個物件在滑鼠下方）
- 3D 物件點擊事件
- 遊戲中的碰撞偵測（子彈、雷射）
- 動畫物件的即時碰撞檢測

> ⚠️ **不要在 `mousemove` 事件中直接投射光線。** 某些瀏覽器觸發 `mousemove` 的頻率可能高於畫面更新頻率，導致效能問題。正確做法：`mousemove` 只更新座標，光線投射在 `update()` 每幀執行一次。

---

## 步驟一：建立 Raycaster.js

`Raycaster` 繼承 `EventEmitter`，在狀態改變時 `trigger` 事件，World 子物件透過 `on` 監聽，不需要混用 `THREE.Object3D` 的 `addEventListener`。

```js
// src/Experience/Raycaster.js
import * as THREE from 'three'
import EventEmitter from './Utils/EventEmitter.js'
import Experience from './Experience.js'

export default class Raycaster extends EventEmitter {
  constructor() {
    super()
    this.experience = new Experience()
    this.camera = this.experience.camera
    this.sizes  = this.experience.sizes

    this.instance = new THREE.Raycaster()
    this.mouse = new THREE.Vector2()
    this.currentIntersect = null
    this.objectsToTest = []

    this._setMouseListeners()
  }

  // 由 World 或子模組呼叫，設定要被測試的物件
  setObjectsToTest(objects) {
    this.objectsToTest = objects
  }

  _setMouseListeners() {
    // 只在 mousemove 裡更新座標，不做光線投射
    this._onMouseMove = (event) => {
      // 標準化至 -1 ~ +1，Y 軸需反向（Three.js Y 軸向上為正）
      this.mouse.x =  (event.clientX / this.sizes.width)  * 2 - 1
      this.mouse.y = -(event.clientY / this.sizes.height) * 2 + 1
    }

    this._onClick = () => {
      if (this.currentIntersect) {
        this.trigger('click', this.currentIntersect)
      }
    }

    window.addEventListener('mousemove', this._onMouseMove)
    window.addEventListener('click', this._onClick)
  }

  // 由 Experience.update() 每幀呼叫
  update() {
    if (this.objectsToTest.length === 0) return

    this.instance.setFromCamera(this.mouse, this.camera.instance)
    const intersects = this.instance.intersectObjects(this.objectsToTest)

    if (intersects.length) {
      if (!this.currentIntersect) {
        // 剛進入（相當於 mouseenter）
        this.trigger('enter', intersects[0])
      }
      this.currentIntersect = intersects[0]
    } else {
      if (this.currentIntersect) {
        // 剛離開（相當於 mouseleave）
        this.trigger('leave', this.currentIntersect)
      }
      this.currentIntersect = null
    }
  }

  destroy() {
    window.removeEventListener('mousemove', this._onMouseMove)
    window.removeEventListener('click', this._onClick)
    this.off('enter')
    this.off('leave')
    this.off('click')
  }
}
```

---

## 步驟二：整合進 Experience.js

```js
// src/Experience/Experience.js
import Raycaster from './Raycaster.js'

export default class Experience {
  constructor(canvas) {
    // ... 其他初始化
    this.raycaster = new Raycaster()   // 在 World 之前建立
    this.world     = new World()
    // ...
  }

  update() {
    this.camera.update()
    this.raycaster.update()   // 在 world.update() 之前，確保 currentIntersect 是最新的
    this.world.update()
    this.renderer.update()
  }

  destroy() {
    // ...
    this.raycaster.destroy()
  }
}
```

---

## 步驟三：在 World 子物件中使用

### 情境 A — Hover 改變顏色

```js
// src/Experience/World/InteractiveMeshes.js
import * as THREE from 'three'
import Experience from '../Experience.js'

export default class InteractiveMeshes {
  constructor() {
    this.experience = new Experience()
    this.scene      = this.experience.scene
    this.raycaster  = this.experience.raycaster

    this._createMeshes()
    this._setRaycaster()
  }

  _createMeshes() {
    const geometry = new THREE.SphereGeometry(0.5)

    this.meshes = [
      new THREE.Mesh(geometry, new THREE.MeshStandardMaterial({ color: '#ff0000' })),
      new THREE.Mesh(geometry, new THREE.MeshStandardMaterial({ color: '#ff0000' })),
      new THREE.Mesh(geometry, new THREE.MeshStandardMaterial({ color: '#ff0000' })),
    ]
    this.meshes[0].position.x = -2
    this.meshes[2].position.x =  2

    this.scene.add(...this.meshes)
  }

  _setRaycaster() {
    this.raycaster.setObjectsToTest(this.meshes)

    this.raycaster.on('enter', (intersect) => {
      intersect.object.material.color.set('#0000ff')
    })
    this.raycaster.on('leave', (intersect) => {
      intersect.object.material.color.set('#ff0000')
    })
    this.raycaster.on('click', (intersect) => {
      console.log('clicked', intersect.object)
    })
  }
}
```

### 情境 B — 點擊不同物件執行不同動作

`trigger` 傳入的 `intersect` 包含 `intersect.object`，透過比對物件參照決定要執行什麼：

```js
_setRaycaster() {
  this.raycaster.setObjectsToTest(this.meshes)

  this.raycaster.on('click', (intersect) => {
    switch (intersect.object) {
      case this.meshes[0]:
        this._openDoor()
        break
      case this.meshes[1]:
        this._playAnimation()
        break
      case this.meshes[2]:
        this._toggleLight()
        break
    }
  })
}
```

### 情境 C — GLTF 模型的 Raycasting

GLTF 模型是巢狀結構，直接傳入 `model`（`gltf.scene`），Raycaster 會遞迴偵測其所有子物件。事件以模型為單位處理，不需要逐一 traverse：

```js
// src/Experience/World/Duck.js
import Experience from '../Experience.js'

export default class Duck {
  constructor() {
    this.experience = new Experience()
    this.scene      = this.experience.scene
    this.resources  = this.experience.resources
    this.raycaster  = this.experience.raycaster

    this._setModel()
  }

  _setModel() {
    const gltf = this.resources.items.duckModel
    this.model = gltf.scene
    this.model.position.y = -1.2
    this.scene.add(this.model)

    this.raycaster.setObjectsToTest([this.model])

    this.raycaster.on('enter', () => {
      this.model.scale.set(1.2, 1.2, 1.2)
    })
    this.raycaster.on('leave', () => {
      this.model.scale.set(1, 1, 1)
    })
  }

  update() {
    // 需要動畫時在此更新
  }
}
```

> ⚠️ **GLTF 模型注意**：`MeshStandardMaterial` 需要燈光才看得見。若模型一片黑，記得在 World 加入 `AmbientLight` 與 `DirectionalLight`。

---

## 情境 D — 動畫物件的碰撞偵測（非滑鼠）

若需要偵測兩個動畫物件之間是否碰撞（例如雷射掃過場景），直接設定光線起點與方向，不需要滑鼠座標。這種情況不使用 EventEmitter，直接在 `update()` 裡處理：

```js
// update() 內，每幀手動設定光線
update() {
  const rayOrigin    = new THREE.Vector3(-3, 0, 0)
  const rayDirection = new THREE.Vector3(1, 0, 0)  // 必須已標準化（長度為 1）
  this.experience.raycaster.instance.set(rayOrigin, rayDirection)

  const intersects = this.experience.raycaster.instance.intersectObjects(this.objectsToTest)

  for (const object of this.objectsToTest) {
    object.material.color.set('#ff0000')
  }
  for (const intersect of intersects) {
    intersect.object.material.color.set('#0000ff')
  }
}
```

> ⚠️ 方向向量**必須標準化**（長度為 1）。若向量長度不為 1，`raycaster.set()` 的結果不可預期。已知長度為 1 時仍建議呼叫 `.normalize()` 以確保安全。

---

## 交集結果欄位

```js
const intersects = raycaster.intersectObjects(objectsToTest)

intersects[0].distance   // 光線原點到碰撞點的距離
intersects[0].point      // 碰撞的確切 3D 座標（Vector3）
intersects[0].object     // 被碰撞的物件（THREE.Mesh）
intersects[0].face       // 被碰撞的面（三角形）
intersects[0].faceIndex  // 面的索引
intersects[0].uv         // 碰撞點的 UV 座標
```

---

## intersectObject vs intersectObjects

| 方法 | 用途 |
|------|------|
| `intersectObject(obj)` | 測試單一物件（含子物件，遞迴） |
| `intersectObjects(arr)` | 測試物件陣列（含子物件，遞迴） |
| `intersectObject(obj, false)` | 測試單一物件（不含子物件） |
| `intersectObjects(arr, false)` | 測試物件陣列（不含子物件） |

兩者都回傳陣列。即使只測試一個物件，也可能有多個交集（例如甜甜圈的前後兩面都可能被射線穿過）。

---

## 常見問題快速排查

| 症狀 | 原因 | 解法 |
|------|------|------|
| 點擊/hover 沒反應 | 未呼叫 `setObjectsToTest()` | 確認 World 子物件有傳入物件陣列 |
| GLTF 模型點擊偏移 | 模型尚未載入就設定測試目標 | 在 `resources.on('ready')` 回調內才呼叫 `setObjectsToTest()` |
| 動畫物件偵測不到 | `setObjectsToTest()` 只呼叫一次，物件已移動 | 測試陣列存的是物件參照，位置更新後不需重新設定 |
| 方向向量光線結果異常 | 向量未標準化 | 呼叫 `.normalize()` |
| 滑鼠 Y 軸偵測上下相反 | 座標轉換公式少了負號 | `mouse.y = -(event.clientY / sizes.height) * 2 + 1` |
| 多個物件共用一個 Raycaster 但需要分組 | 所有物件共用同一個 `setObjectsToTest` | 視需求建立多個 Raycaster 實例，或在 `update()` 裡手動分組測試 |
| GLTF 模型一片黑 | MeshStandardMaterial 需要燈光 | 場景加入 AmbientLight 與 DirectionalLight |
