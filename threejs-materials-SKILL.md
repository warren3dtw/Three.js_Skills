---
name: threejs-materials
description: >
  Three.js 材質（Material）選擇、屬性設定與紋理整合的完整指引。當使用者在 Three.js 專案中涉及任何材質相關操作時，請務必使用此技能，包括：選擇適合的 Material 類型、設定 map / color / wireframe / opacity / alphaMap 等屬性、整合 PBR 材質（MeshStandardMaterial / MeshPhysicalMaterial）、載入環境貼圖（RGBELoader）、使用 MeshMatcapMaterial 或 MeshToonMaterial、處理 metalness / roughness / normalMap 等進階屬性。即使使用者只說「材質」、「質感」、「貼圖」、「反射」、「透明」，都應觸發此技能。
---

# Three.js Materials Skill

當使用者需要在 Three.js 中設定材質時，依照以下流程進行。

---

## 步驟一：選擇材質類型

根據需求從下表選擇材質，並告知使用者選擇原因：

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
| `MeshPhysicalMaterial` | ✅ | PBR 進階材質，支援 clearcoat / sheen / iridescence / transmission，效能最差 |

> ⚠️ `MeshLambertMaterial` 以後的材質需要燈光才能顯示，場景中若無光源會全黑。

---

## 步驟二：建立材質與場景測試物件

```js
// 載入紋理（在建立 material 之前）
const textureLoader = new THREE.TextureLoader()
const doorColorTexture = textureLoader.load('./textures/door/color.jpg')
doorColorTexture.colorSpace = THREE.SRGBColorSpace  // map / matcap 類型必須設定

// 建立測試場景（球 / 平面 / 圓環）
const material = new THREE.MeshStandardMaterial()

const sphere = new THREE.Mesh(new THREE.SphereGeometry(0.5, 16, 16), material)
sphere.position.x = -1.5

const plane = new THREE.Mesh(new THREE.PlaneGeometry(1, 1), material)

const torus = new THREE.Mesh(new THREE.TorusGeometry(0.3, 0.2, 16, 32), material)
torus.position.x = 1.5

scene.add(sphere, plane, torus)  // scene.add() 支援一次傳入多個物件
```

---

## 步驟三：設定各材質屬性

### MeshBasicMaterial

```js
const material = new THREE.MeshBasicMaterial()
material.map = doorColorTexture              // 套用紋理
material.color = new THREE.Color('#ff0000')  // 顏色（支援 '#ff0000' / '#f00' / 'red' / 'rgb(255,0,0)' / 0xff0000）
material.wireframe = true                    // 顯示三角面線框
material.transparent = true
material.opacity = 0.5                       // 透明度（需先設 transparent: true）
material.alphaMap = doorAlphaTexture         // 用紋理控制透明區域
material.side = THREE.DoubleSide             // FrontSide（預設）/ BackSide / DoubleSide
```

> ⚠️ `THREE.DoubleSide` 消耗較多運算資源，僅在必要時使用。`wireframe` 和 `opacity` 屬性在多數材質通用。

### MeshNormalMaterial

```js
const material = new THREE.MeshNormalMaterial()
material.flatShading = true  // 讓每個面保持獨立法線，呈現稜角感
// 顏色反映法線方向相對於鏡頭的角度，繞著物體轉顏色不會改變
```

### MeshMatcapMaterial

```js
const matcapTexture = textureLoader.load('./textures/matcaps/1.png')
matcapTexture.colorSpace = THREE.SRGBColorSpace

const material = new THREE.MeshMatcapMaterial()
material.matcap = matcapTexture
// 光照為假象，不受燈光影響，效能極佳
```

> 免費 matcap 資源：https://github.com/nidorx/matcaps
> 線上製作工具：https://www.kchapelier.com/matcap-studio/

### MeshDepthMaterial

```js
const material = new THREE.MeshDepthMaterial()
// 靠近 near 的物體呈白色，靠近 far 的物體呈黑色
// 主要用途：為後續陰影計算儲存深度資訊
```

### 需要燈光的材質：先加入燈光

```js
const ambientLight = new THREE.AmbientLight(0xffffff, 1)
scene.add(ambientLight)

const pointLight = new THREE.PointLight(0xffffff, 30)
pointLight.position.set(2, 3, 4)
scene.add(pointLight)
```

### MeshLambertMaterial

```js
const material = new THREE.MeshLambertMaterial()
// 效能最佳的光照材質，但在圓弧幾何體邊緣可見光照瑕疵
```

### MeshPhongMaterial

```js
const material = new THREE.MeshPhongMaterial()
material.shininess = 100                       // 高光強度（數值越高越亮）
material.specular = new THREE.Color(0x1188ff)  // 高光顏色
```

### MeshToonMaterial

```js
const gradientTexture = textureLoader.load('./textures/gradients/3.jpg')
// 小尺寸漸層貼圖需設定 NearestFilter，否則 GPU 插值會破壞卡通效果
gradientTexture.minFilter = THREE.NearestFilter
gradientTexture.magFilter = THREE.NearestFilter
gradientTexture.generateMipmaps = false  // NearestFilter 不需要 mipmaps，關閉節省記憶體

const material = new THREE.MeshToonMaterial()
material.gradientMap = gradientTexture
```

### MeshStandardMaterial（推薦）

```js
const material = new THREE.MeshStandardMaterial()
material.metalness = 0.7    // 金屬感（0~1）
material.roughness = 0.2    // 粗糙度（0~1）

// 紋理貼圖
material.map = doorColorTexture
material.aoMap = doorAmbientOcclusionTexture     // 環境遮蔽，暗部增強立體感
material.aoMapIntensity = 1
material.displacementMap = doorHeightTexture     // 實際移動頂點產生凹凸
material.displacementScale = 0.1                // 位移強度（預設過強，調小）
material.metalnessMap = doorMetalnessTexture     // 用紋理控制金屬感分佈
material.roughnessMap = doorRoughnessTexture     // 用紋理控制粗糙度分佈
material.normalMap = doorNormalTexture           // 假法線細節（不移動頂點）
material.normalScale.set(0.5, 0.5)              // 法線強度（Vector2）
material.transparent = true
material.alphaMap = doorAlphaTexture
```

> ⚠️ 同時使用 `metalnessMap` / `roughnessMap` 時，需將 `metalness` 和 `roughness` 設為 `1`，否則純量值會與貼圖互相干擾。

> ⚠️ `displacementMap` 需要足夠頂點才有效果，建議提高幾何體細分數：
> ```js
> new THREE.SphereGeometry(0.5, 64, 64)
> new THREE.PlaneGeometry(1, 1, 100, 100)
> new THREE.TorusGeometry(0.3, 0.2, 64, 128)
> ```

### 環境貼圖（搭配 Standard / Physical / Lambert / Phong）

```js
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader.js'

const rgbeLoader = new RGBELoader()
rgbeLoader.load('./textures/environmentMap/2k.hdr', (environmentMap) => {
  environmentMap.mapping = THREE.EquirectangularReflectionMapping
  scene.background = environmentMap   // 設定場景背景
  scene.environment = environmentMap  // 套用到所有支援的材質
})
// 注意：RGBELoader 與 TextureLoader 不同，需傳入 callback，在 callback 中使用 environmentMap
```

### MeshPhysicalMaterial（MeshStandardMaterial 的進階版）

繼承 Standard 所有屬性，額外支援以下效果（逐一測試，不同時啟用）：

```js
const material = new THREE.MeshPhysicalMaterial()
// 基礎屬性同 MeshStandardMaterial...

// Clearcoat：模擬表面光漆層（如汽車烤漆）
material.clearcoat = 1
material.clearcoatRoughness = 0

// Sheen：布料感，窄角度下出現光暈
material.sheen = 1
material.sheenRoughness = 0.25
material.sheenColor.set(1, 1, 1)

// Iridescence：彩虹油膜效果（燃料水坑、肥皂泡）
material.iridescence = 1
material.iridescenceIOR = 1
material.iridescenceThicknessRange = [100, 800]

// Transmission：透光折射（比 opacity 更真實，背景圖像會扭曲）
material.transmission = 1
material.ior = 1.5      // 折射率（鑽石 2.417 / 水 1.333 / 空氣 1.000293）
material.thickness = 0.5
```

> ⚠️ `MeshPhysicalMaterial` 效能最差，避免用於大量物件或低效能裝置。

---

## 步驟四：加入 Debug UI（開發階段）

```js
// npm install lil-gui
import GUI from 'lil-gui'

const gui = new GUI()
gui.add(material, 'metalness').min(0).max(1).step(0.0001)
gui.add(material, 'roughness').min(0).max(1).step(0.0001)
gui.add(material, 'wireframe')
gui.addColor(material, 'color')
```

---

## 常見問題快速排查

| 症狀 | 原因 | 解法 |
|------|------|------|
| 場景全黑（無報錯） | 使用需要燈光的材質但場景無光源 | 加入 `AmbientLight` + `PointLight`，或使用環境貼圖 |
| 貼圖顏色偏暗或不對 | 未設定 colorSpace | `texture.colorSpace = THREE.SRGBColorSpace`（map / matcap 類型需設定） |
| MeshToonMaterial 卡通效果消失 | 漸層貼圖被 GPU 插值模糊 | 設定 `minFilter / magFilter = THREE.NearestFilter` |
| displacementMap 效果太誇張 | 預設強度過高或頂點不足 | 調低 `displacementScale`，並增加幾何體細分數 |
| metalnessMap 看起來怪異 | metalness / roughness 純量值干擾貼圖 | `material.metalness = 1; material.roughness = 1` |
| 透明效果無效 | 未開啟 transparent | `material.transparent = true` |
| aoMap 無效果 | aoMap 只受特定光源影響 | aoMap 僅對 `AmbientLight`、環境貼圖、`HemisphereLight` 有效 |
