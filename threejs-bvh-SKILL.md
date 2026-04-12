---
name: threejs-bvh
description: >
  當使用者在 Three.js 專案中涉及高效能光線投射、大量物件的碰撞偵測、
  點到網格的最近距離查詢、或需要加速 Raycaster 效能時，請使用此 skill。
  觸發情境包括：場景中有大量三角形導致 raycaster 變慢、需要精確的網格 vs 網格碰撞、
  動畫骨架網格的 BVH 更新、批次幾何體（BatchedMesh）的射線檢測、
  序列化 BVH 到 Worker 非同步建構。
  即使使用者沒有說「BVH」，只要提到「raycaster 很慢」、「大量物件碰撞」、
  「加速射線」、「網格碰撞優化」，都應觸發此 skill。
  本技能以 threejs-experience 的 Experience 架構為前提，
  BVH 初始化掛載於 Experience.js，查詢邏輯在各 World 子模組或 Raycaster.js 中進行。
---

# Three.js BVH Skill

本技能以 **Experience Singleton 架構**（`threejs-experience`）為前提。  
`three-mesh-bvh` 透過將 BVH 功能掛載到 Three.js prototype，讓原本的 `THREE.Raycaster` 自動使用加速結構，不需改寫現有的 raycaster 邏輯。

---

## 安裝

```bash
npm install three-mesh-bvh
```

---

## 步驟一：在 Experience.js 掛載 BVH prototype

BVH prototype 掛載必須在任何幾何體建立前執行一次，放在 `Experience.js` 頂層：

```js
// src/Experience/Experience.js
import * as THREE from 'three'
import {
  computeBoundsTree,
  disposeBoundsTree,
  acceleratedRaycast,
} from 'three-mesh-bvh'

// 掛載到 Three.js prototype（全局一次，必須在 Scene / Geometry 初始化前執行）
THREE.BufferGeometry.prototype.computeBoundsTree = computeBoundsTree
THREE.BufferGeometry.prototype.disposeBoundsTree = disposeBoundsTree
THREE.Mesh.prototype.raycast = acceleratedRaycast

let instance = null

export default class Experience {
  constructor(canvas) {
    if (instance) return instance
    instance = this
    // ... 其餘初始化
  }
}
```

> ⚠️ 掛載只需執行一次。Singleton 架構保證 `Experience.js` 的頂層只執行一次，因此放在此處是最安全的位置。

---

## 步驟二：在 World 子物件中建構 BVH

幾何體建立後，呼叫 `computeBoundsTree()` 即可讓該幾何體的所有 raycast 自動走 BVH 加速路徑。

```js
// src/Experience/World/StaticLevel.js
import * as THREE from 'three'
import Experience from '../Experience.js'

export default class StaticLevel {
  constructor() {
    this.experience = new Experience()
    this.scene      = this.experience.scene
    this.resources  = this.experience.resources

    this._setGeometry()
    this._setMaterial()
    this._setMesh()
  }

  _setGeometry() {
    this.geometry = new THREE.TorusKnotGeometry(1, 0.3, 256, 32)
    // 建構 BVH（掛載 prototype 後直接可用）
    this.geometry.computeBoundsTree()
  }

  _setMaterial() {
    this.material = new THREE.MeshStandardMaterial({ color: '#888888' })
  }

  _setMesh() {
    this.mesh = new THREE.Mesh(this.geometry, this.material)
    this.scene.add(this.mesh)
  }

  destroy() {
    // 釋放 BVH 佔用的記憶體
    this.geometry.disposeBoundsTree()
    this.geometry.dispose()
    this.material.dispose()
  }
}
```

---

## 步驟三：在 Raycaster.js 啟用 firstHitOnly

若與 `threejs-raycaster` skill 搭配使用，在 `Raycaster.js` 的 `constructor` 中加入：

```js
// src/Experience/Raycaster.js
constructor() {
  // ...
  this.instance = new THREE.Raycaster()
  this.instance.firstHitOnly = true  // 只尋找最近命中，BVH 加速下效能最佳
  // ...
}
```

> `firstHitOnly = true` 讓 BVH 找到第一個命中後立即停止遍歷，適用於大多數滑鼠互動場景。若需要所有命中（例如穿透物件），移除此設定。

---

## BVH 建構策略選擇

| 策略 | 建構速度 | 查詢速度 | 適用情境 |
|------|---------|---------|---------|
| `CENTER`（預設） | 快 | 普通 | 動態頻繁更新的幾何體 |
| `AVERAGE` | 普通 | 普通 | 一般用途 |
| `SAH` | 慢 | 最快 | 靜態資源，查詢頻率高 |

靜態場景（地形、建築）使用 `SAH`：

```js
import { MeshBVH, SAH } from 'three-mesh-bvh'

this.geometry.boundsTree = new MeshBVH(this.geometry, {
  strategy: SAH,
  maxDepth: 40,
  maxLeafSize: 10,
})
```

---

## 距離查詢

查詢一個世界座標點到網格表面的最近距離：

```js
// src/Experience/World/ProximityChecker.js
update() {
  const bvh = this.mesh.geometry.boundsTree
  if (!bvh) return

  // 將世界座標轉換到網格的 local space
  const localPoint = this.targetPoint.clone()
    .applyMatrix4(new THREE.Matrix4().copy(this.mesh.matrixWorld).invert())

  const result = bvh.closestPointToPoint(localPoint, {}, 0, Infinity)
  if (result) {
    console.log('最近距離:', result.distance)
    console.log('命中點 (local):', result.point)
    // 轉回世界座標
    result.point.applyMatrix4(this.mesh.matrixWorld)
  }
}
```

> ⚠️ BVH 直接查詢時，**所有座標必須是 local space**。使用 `mesh.matrixWorld` 的反矩陣轉換輸入，結果也需用 `matrixWorld` 轉回世界座標。

---

## 碰撞偵測

### 網格 vs Box / Sphere

```js
const bvh = this.mesh.geometry.boundsTree

// vs Box（local space）
const box = new THREE.Box3().setFromObject(otherObject)
const hit = bvh.intersectsBox(box, this.mesh.matrixWorld)

// vs Sphere（local space）
const sphere = new THREE.Sphere(center, radius)
const hit = bvh.intersectsSphere(sphere)
```

### 網格 vs 網格（兩個 BVH 互測）

```js
// src/Experience/World/CollisionPair.js
_checkCollision() {
  const bvh1 = this.meshA.geometry.boundsTree
  const bvh2 = this.meshB.geometry.boundsTree

  // 將 meshB 的 BVH 轉換到 meshA 的 local space
  const matrixToLocal = new THREE.Matrix4()
    .copy(this.meshB.matrixWorld)
    .premultiply(new THREE.Matrix4().copy(this.meshA.matrixWorld).invert())

  const isColliding = bvh1.intersectsGeometry(this.meshB.geometry, matrixToLocal)
  return isColliding
}
```

---

## 動態幾何體（refit）

幾何體頂點位置變動後，不需重建 BVH，只需 `refit()`（比重建快得多）：

```js
// update() 內
update() {
  // 修改頂點...
  this.positionAttribute.needsUpdate = true
  this.geometry.computeVertexNormals()

  // 同步 BVH（不重建，效能友好）
  this.geometry.boundsTree.refit()
}
```

---

## 骨骼動畫網格（SkinnedMesh）

```js
import { SkinnedMeshBVH, CENTER } from 'three-mesh-bvh'

// 建構
const bvh = new SkinnedMeshBVH(this.skinnedMesh, { strategy: CENTER })
this.skinnedMesh.geometry.boundsTree = bvh

// update() 內，動畫更新後同步 BVH
update() {
  this.mixer.update(this.experience.time.delta * 0.001)
  bvh.refit()
}
```

---

## Worker 非同步建構（大型模型）

對於載入的 GLTF 大型模型，避免主執行緒卡頓：

```js
// src/Experience/World/LargeModel.js
import { ParallelMeshBVHWorker } from 'three-mesh-bvh/worker'

_setBVH() {
  const worker = new ParallelMeshBVHWorker()

  this.mesh.geometry.traverse?.((child) => {
    if (child.isMesh) {
      worker.generate(child.geometry).then(bvh => {
        child.geometry.boundsTree = bvh
      })
    }
  })
}
```

---

## 調試可視化（搭配 Debug 模組）

在 `#debug` 模式下顯示 BVH 層級結構：

```js
import { BVHHelper } from 'three-mesh-bvh'

constructor() {
  // ...
  this.debug = this.experience.debug
  this._setDebug()
}

_setDebug() {
  if (!this.debug.active) return

  this.bvhHelper = new BVHHelper(this.mesh)
  this.bvhHelper.depth = 10  // 顯示到第幾層節點
  this.scene.add(this.bvhHelper)
  this.bvhHelper.update()

  const folder = this.debug.ui.addFolder('BVH Helper')
  folder.add(this.bvhHelper, 'depth').min(1).max(20).step(1)
    .onChange(() => this.bvhHelper.update())
}

destroy() {
  this.bvhHelper?.dispose()
  this.scene.remove(this.bvhHelper)
}
```

---

## 效能技巧

| 技巧 | 說明 |
|------|------|
| `firstHitOnly = true` | 找到第一個命中即停止，滑鼠互動場景必開 |
| 靜態資源用 `SAH` | 建構慢但查詢最快，適合場景地形 |
| 動態資源用 `CENTER` | 建構快，適合頻繁 refit |
| `geometry.center()` | 幾何體中心偏移很大時避免浮點精度問題 |
| `indirect: true` | 保留原始 index 排列（BatchedMesh 必須） |
| 設定 `raycaster.near / far` | 縮小搜尋範圍，減少 BVH 遍歷 |

---

## 常見問題快速排查

| 症狀 | 原因 | 解法 |
|------|------|------|
| BVH 加速無效，raycaster 仍然慢 | prototype 未掛載或幾何體先於掛載建立 | 確認 import 與掛載在 `Experience.js` 頂層，所有幾何體建立於此之後 |
| 命中點位置錯誤 | 直接查詢 BVH 時使用了世界座標 | 輸入前用 `matrixWorld` 反矩陣轉換至 local space |
| 動畫後碰撞偵測失效 | 頂點更新後未呼叫 refit | `update()` 內在頂點修改後呼叫 `geometry.boundsTree.refit()` |
| GLTF 模型掛載後找不到 boundsTree | 模型的每個子 Mesh 需個別 computeBoundsTree | 用 `mesh.traverse` 對每個 `child.isMesh` 呼叫 |
| 頁面卸載後記憶體未釋放 | 未呼叫 disposeBoundsTree | `destroy()` 中呼叫 `geometry.disposeBoundsTree()` |
| SkinnedMesh 碰撞偵測不準 | 使用了一般 MeshBVH 而非 SkinnedMeshBVH | 改用 `new SkinnedMeshBVH(skinnedMesh)` |
