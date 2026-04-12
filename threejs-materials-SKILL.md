---
name: threejs-materials
description: >
  Three.js 材質（Material）選擇、屬性設定與紋理整合的完整指引。當使用者在 Three.js 專案中涉及任何材質相關操作時，請務必使用此技能，包括：選擇適合的 Material 類型、設定 map / color / wireframe / opacity / alphaMap 等屬性、整合 PBR 材質（MeshStandardMaterial / MeshPhysicalMaterial）、載入環境貼圖（RGBELoader）、使用 MeshMatcapMaterial 或 MeshToonMaterial、處理 metalness / roughness / normalMap 等進階屬性。即使使用者只說「材質」、「質感」、「貼圖」、「反射」、「透明」，都應觸發此技能。
  本技能以 threejs-experience 的 Experience 架構為前提，所有材質操作都在 World 子模組內進行。
---

# Three.js Materials Skill

本技能以 **Experience Singleton 架構**（`threejs-experience`）為前提。材質邏輯集中在 `src/Experience/World/` 下的各個物件模組，透過 `new Experience()` 取得 singleton 後存取 `resources`、`scene`、`debug`。

---

## 架構中的材質定位

材質的建立遵循 World 物件的固定模式，拆成獨立方法：

```js
// 固定模式
_setGeometry()   // 建立幾何體
_setTextures()   // 從 resources.items 取得紋理，設定 colorSpace / repeat 等
_setMaterial()   // 建立材質，套用紋理
_setMesh()       // 建立 Mesh，加入 scene
```

紋理資源透過 `Resources` 統一載入，**不在 World 物件內自行使用 `TextureLoader`**。

---

## 步驟一：在 sources.js 宣告紋理資源

所有紋理先在資源清單中登記：

```js
// src/Experience/sources.js
export default [
  { name: 'ColorTexture',             type: 'texture', path: 'textures/color.jpg' },
  { name: 'AlphaTexture',             type: 'texture', path: 'textures/alpha.jpg' },
  { name: 'AmbientOcclusionTexture',  type: 'texture', path: 'textures/ambientOcclusion.jpg' },
  { name: 'HeightTexture',            type: 'texture', path: 'textures/height.jpg' },
  { name: 'NormalTexture',            type: 'texture', path: 'textures/normal.jpg' },
  { name: 'MetalnessTexture',         type: 'texture', path: 'textures/metalness.jpg' },
  { name: 'RoughnessTexture',         type: 'texture', path: 'textures/roughness.jpg' },
  { name: 'matcapTexture',                type: 'texture', path: 'textures/matcaps/1.png' },
  { name: 'gradientTexture',              type: 'texture', path: 'textures/gradients/3.jpg' },
]
```

---

## 步驟二：選擇材質類型

根據需求從下表選擇，並告知使用者選擇原因：

| 材質 | 需要燈光 | 適用情境 |
|------|----------|----------|
| `MeshBasicMaterial` | ❌ | 純色/貼圖，無光影，最輕量 |
| `MeshNormalMaterial` | ❌ | 法線視覺化，除錯用途 |
| `MeshMatcapMaterial` | ❌ | 高品質假光照，效能極佳 |
| `MeshDepthMaterial` | ❌ | 深度視覺化，陰影計算用 |
| `MeshLambertMaterial` | ✅ | 光照材質中效能最佳，但邊緣有瑕疵 |
| `MeshPhongMaterial` | ✅ | 有高光反射，比 Lambert 稍慢 |
| `MeshToonMaterial` | ✅ | 卡通風格（cel shading） |
| `MeshStandardMaterial` | ✅ | PBR 標準材質，**最推薦的通用選擇** |
| `MeshPhysicalMaterial` | ✅ | PBR 進階，支援 clearcoat / sheen / iridescence / transmission，效能最差 |

> ⚠️ `MeshLambertMaterial` 以後的材質需要燈光才能顯示。若場景無光源，改用環境貼圖或在 `Environment.js` 加入燈光。

---

## 步驟三：在 World 物件中建立材質

以下展示各材質在 singleton 架構下的完整寫法。

### MeshBasicMaterial

```js
// src/Experience/World/TestObject.js
import * as THREE from 'three'
import Experience from '../Experience.js'

export default class TestObject {
  constructor() {
    this.experience = new Experience()
    this.scene     = this.experience.scene
    this.resources = this.experience.resources

    this._setGeometry()
    this._setTextures()
    this._setMaterial()
    this._setMesh()
  }

  _setGeometry() {
    this.geometry = new THREE.BoxGeometry(1, 1, 1)
  }

  _setTextures() {
    this.textures = {}
    this.textures.color = this.resources.items.doorColorTexture
    this.textures.color.colorSpace = THREE.SRGBColorSpace  // map 類型必須設定
  }

  _setMaterial() {
    this.material = new THREE.MeshBasicMaterial({
      map: this.textures.color,
    })
    // 其他可用屬性：
    // this.material.color = new THREE.Color('#ff0000')
    // this.material.wireframe = true
    // this.material.transparent = true
    // this.material.opacity = 0.5
    // this.material.alphaMap = this.resources.items.doorAlphaTexture
    // this.material.side = THREE.DoubleSide  // 慎用，消耗較多資源
  }

  _setMesh() {
    this.mesh = new THREE.Mesh(this.geometry, this.material)
    this.scene.add(this.mesh)
  }
}
```

### MeshMatcapMaterial

```js
_setTextures() {
  this.textures = {}
  this.textures.matcap = this.resources.items.matcapTexture
  this.textures.matcap.colorSpace = THREE.SRGBColorSpace  // matcap 需設定 colorSpace
}

_setMaterial() {
  this.material = new THREE.MeshMatcapMaterial({
    matcap: this.textures.matcap,
  })
}
```


### MeshNormalMaterial

```js
_setMaterial() {
  this.material = new THREE.MeshNormalMaterial()
  this.material.flatShading = true  // 讓每個面保持獨立法線，呈現稜角感
}
```

### MeshToonMaterial

```js
_setTextures() {
  this.textures = {}
  this.textures.gradient = this.resources.items.gradientTexture
  // 小尺寸漸層貼圖需設定 NearestFilter，否則 GPU 插值會破壞卡通效果
  this.textures.gradient.minFilter = THREE.NearestFilter
  this.textures.gradient.magFilter = THREE.NearestFilter
  this.textures.gradient.generateMipmaps = false  // NearestFilter 不需要 mipmaps
}

_setMaterial() {
  this.material = new THREE.MeshToonMaterial({
    gradientMap: this.textures.gradient,
  })
}
```

### MeshStandardMaterial（推薦）

```js
_setTextures() {
  this.textures = {}

  this.textures.color = this.resources.items.doorColorTexture
  this.textures.color.colorSpace = THREE.SRGBColorSpace

  this.textures.ao            = this.resources.items.doorAmbientOcclusionTexture
  this.textures.height        = this.resources.items.doorHeightTexture
  this.textures.normal        = this.resources.items.doorNormalTexture
  this.textures.metalness     = this.resources.items.doorMetalnessTexture
  this.textures.roughness     = this.resources.items.doorRoughnessTexture
  this.textures.alpha         = this.resources.items.doorAlphaTexture
}

_setMaterial() {
  this.material = new THREE.MeshStandardMaterial()
  this.material.map                = this.textures.color
  this.material.aoMap              = this.textures.ao        // 環境遮蔽，暗部增強立體感
  this.material.aoMapIntensity     = 1
  this.material.displacementMap    = this.textures.height    // 實際移動頂點產生凹凸
  this.material.displacementScale  = 0.1                    // 強度（預設過強，調小）
  this.material.metalnessMap       = this.textures.metalness
  this.material.roughnessMap       = this.textures.roughness
  this.material.metalness          = 1   // 使用 map 時需設為 1，避免與純量互相干擾
  this.material.roughness          = 1   // 同上
  this.material.normalMap          = this.textures.normal
  this.material.normalScale.set(0.5, 0.5)  // 法線強度（Vector2）
  this.material.transparent        = true
  this.material.alphaMap           = this.textures.alpha
}

_setMesh() {
  // displacementMap 需要足夠頂點，提高幾何體細分數
  this.geometry = new THREE.SphereGeometry(0.5, 64, 64)
  this.mesh = new THREE.Mesh(this.geometry, this.material)
  this.scene.add(this.mesh)
}
```

### 環境貼圖（HDR，搭配 Standard / Physical）

環境貼圖通常在 `Environment.js` 中處理，使用 `RGBELoader`（需額外引入，不在 Resources 內建 loaders 中）：

```js
// src/Experience/World/Environment.js
import * as THREE from 'three'
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader.js'
import Experience from '../Experience.js'

export default class Environment {
  constructor() {
    this.experience = new Experience()
    this.scene  = this.experience.scene
    this.debug  = this.experience.debug

    this._setEnvironmentMap()
  }

  _setEnvironmentMap() {
    const rgbeLoader = new RGBELoader()
    rgbeLoader.load('./textures/environmentMap/2k.hdr', (environmentMap) => {
      environmentMap.mapping = THREE.EquirectangularReflectionMapping
      this.scene.background  = environmentMap
      this.scene.environment = environmentMap  // 套用到所有 Standard / Physical 材質
    })
  }
}
```

> ⚠️ `RGBELoader` 與 `TextureLoader` 不同，需傳入 callback，在 callback 中才能使用載入完的 `environmentMap`。

### MeshPhysicalMaterial（MeshStandardMaterial 進階版）

繼承 Standard 所有屬性，額外支援下列效果（逐一測試，不同時啟用）：

```js
_setMaterial() {
  this.material = new THREE.MeshPhysicalMaterial()
  // 繼承 Standard 的基礎屬性...
  this.material.metalness = 0
  this.material.roughness = 0

  // Clearcoat：模擬表面光漆層（如汽車烤漆）
  this.material.clearcoat = 1
  this.material.clearcoatRoughness = 0

  // Sheen：布料感，窄角度下出現光暈
  this.material.sheen = 1
  this.material.sheenRoughness = 0.25
  this.material.sheenColor.set(1, 1, 1)

  // Iridescence：彩虹油膜效果（燃料水坑、肥皂泡）
  this.material.iridescence = 1
  this.material.iridescenceIOR = 1
  this.material.iridescenceThicknessRange = [100, 800]

  // Transmission：透光折射（比 opacity 更真實）
  this.material.transmission = 1
  this.material.ior = 1.5       // 折射率（鑽石 2.417 / 水 1.333 / 空氣 1.000293）
  this.material.thickness = 0.5
}
```

> ⚠️ `MeshPhysicalMaterial` 效能最差，避免用於大量物件或低效能裝置。

---

## 步驟四：加入 Debug UI（選用）

在 World 物件中透過 `debug` 模組加入調整面板，網址加 `#debug` 才會顯示：

```js
constructor() {
  // ...
  this.debug = this.experience.debug

  this._setGeometry()
  this._setTextures()
  this._setMaterial()
  this._setMesh()
  this._setDebug()
}

_setDebug() {
  if (!this.debug.active) return

  this.debugFolder = this.debug.ui.addFolder('Material')
  this.debugFolder.add(this.material, 'metalness').min(0).max(1).step(0.0001)
  this.debugFolder.add(this.material, 'roughness').min(0).max(1).step(0.0001)
  this.debugFolder.add(this.material, 'wireframe')
  this.debugFolder.addColor(this.material, 'color')
}
```

---

## 需要燈光時：在 World.js 建立後加入 Environment

若使用需要燈光的材質（Lambert / Phong / Toon / Standard / Physical），確認 `World.js` 中有建立 `Environment`：

```js
// src/Experience/World/World.js
this.resources.on('ready', () => {
  this.testObject  = new TestObject()   // ⚠️ Mesh 必須在 Environment 之前建立
  this.environment = new Environment()  // Environment 會遍歷 scene 套用 envMap
})
```

---

## 常見問題快速排查

| 症狀 | 原因 | 解法 |
|------|------|------|
| 材質全黑（無報錯） | 使用需要燈光的材質但場景無光源 | 在 `Environment.js` 加入燈光或環境貼圖 |
| 貼圖顏色偏暗或不對 | 未設定 colorSpace | `texture.colorSpace = THREE.SRGBColorSpace`（map / matcap 類型需設定） |
| `resources.items.xxx` 是 undefined | 資源尚未載入完成 | 確認程式碼在 `resources.on('ready', ...)` callback 內執行 |
| MeshToonMaterial 卡通效果消失 | 漸層貼圖被 GPU 插值模糊 | 設定 `minFilter / magFilter = THREE.NearestFilter` |
| displacementMap 效果太誇張 | 強度過高或頂點不足 | 調低 `displacementScale`，並增加幾何體細分數 |
| metalnessMap 效果怪異 | metalness / roughness 純量值干擾貼圖 | `material.metalness = 1; material.roughness = 1` |
| 透明效果無效 | 未開啟 transparent | `material.transparent = true` |
| Environment 套用 envMap 後部分物件沒反應 | Mesh 在 Environment 之後建立 | 調整 `World.js` 中的建立順序，Mesh 先於 Environment |
