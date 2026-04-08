---
name: threejs-3d-text
description: >
  Three.js 3D 文字建立、置中與場景裝飾的完整指引。當使用者在 Three.js 專案中需要顯示 3D 文字時，請務必使用此技能，包括：載入 typeface 字型、使用 FontLoader 與 TextGeometry、設定文字幾何體參數（size / depth / bevelEnabled）、將文字置中（bounding box / center()）、搭配 MeshMatcapMaterial 套用材質、以及在場景中加入裝飾物件（如漂浮甜甜圈）。即使使用者只說「3D 文字」、「立體字」、「文字幾何體」，都應觸發此技能。
---

# Three.js 3D Text Skill

當使用者需要在 Three.js 中建立 3D 文字時，依照以下流程進行。

---

## 步驟一：準備字型檔案

Three.js 的 `TextGeometry` 需要 typeface JSON 格式的字型，有三種取得方式：

**方式 A：使用 Three.js 內建字型（推薦）**
將 `/node_modules/three/examples/fonts/` 中的 `.typeface.json` 與 `LICENSE` 複製到 `/static/fonts/` 資料夾。

**方式 B：轉換自訂字型**
上傳字型至 https://gero3.github.io/facetype.js/ 轉換為 typeface JSON。

**方式 C：直接在 JS 中 import（Vite 專案）**
```js
import typefaceFont from 'three/examples/fonts/helvetiker_regular.typeface.json'
```

---

## 步驟二：引入必要類別並載入字型

`FontLoader` 和 `TextGeometry` 皆不在 `THREE` 命名空間，需獨立引入：

```js
import { FontLoader } from 'three/examples/jsm/loaders/FontLoader.js'
import { TextGeometry } from 'three/examples/jsm/geometries/TextGeometry.js'
```

**載入字型（非同步，所有依賴字型的程式碼必須在 callback 內）：**

```js
const fontLoader = new FontLoader()

fontLoader.load('/fonts/helvetiker_regular.typeface.json', (font) => {
  // 所有文字相關程式碼寫在這裡
})
```

---

## 步驟三：建立文字幾何體

```js
fontLoader.load('/fonts/helvetiker_regular.typeface.json', (font) => {

  const textGeometry = new TextGeometry('Hello Three.js', {
    font: font,
    size: 0.5,            // 文字大小
    depth: 0.2,           // 文字厚度
    curveSegments: 12,    // 曲線細分數（影響效能，可調低）
    bevelEnabled: true,   // 啟用斜角
    bevelThickness: 0.03,
    bevelSize: 0.02,
    bevelOffset: 0,
    bevelSegments: 5      // 斜角細分數（影響效能，可調低）
  })

  const material = new THREE.MeshBasicMaterial()
  const text = new THREE.Mesh(textGeometry, material)
  scene.add(text)
})
```

> ⚠️ 建立 `TextGeometry` 的計算成本很高。避免頻繁重建，並盡量降低 `curveSegments` 和 `bevelSegments`。可先設 `wireframe: true` 確認細分程度是否合適。

---

## 步驟四：文字置中

**快速方式（推薦）：**
```js
textGeometry.center()
```

**手動方式（可學習 Bounding Box 概念）：**
```js
// 1. 請 Three.js 計算 Box Bounding（預設為 Sphere Bounding）
textGeometry.computeBoundingBox()

// 2. 依 boundingBox.max 平移整個幾何體（非 Mesh）
textGeometry.translate(
  -(textGeometry.boundingBox.max.x - 0.02) * 0.5,  // 減去 bevelSize
  -(textGeometry.boundingBox.max.y - 0.02) * 0.5,  // 減去 bevelSize
  -(textGeometry.boundingBox.max.z - 0.03) * 0.5   // 減去 bevelThickness
)
```

> **Bounding 補充**：Bounding 記錄幾何體佔用的空間（Box 或 Sphere）。Three.js 用它做 Frustum Culling——判斷物體是否在鏡頭視錐內，不在的話直接跳過渲染以節省效能。

---

## 步驟五：套用 MeshMatcapMaterial

```js
const textureLoader = new THREE.TextureLoader()
const matcapTexture = textureLoader.load('/textures/matcaps/1.png')
matcapTexture.colorSpace = THREE.SRGBColorSpace  // matcap 需設定 colorSpace

const material = new THREE.MeshMatcapMaterial({ matcap: matcapTexture })
```

> 免費 matcap 資源：https://github.com/nidorx/matcaps（256x256 解析度已足夠）

---

## 步驟六：加入裝飾物件（選用）

在 fontLoader callback 內，共用 geometry 和 material 建立漂浮裝飾：

```js
// geometry 和 material 移到迴圈外共用，避免重複建立（效能優化關鍵）
const donutGeometry = new THREE.TorusGeometry(0.3, 0.2, 20, 45)
// material 直接與文字共用同一個實例

for (let i = 0; i < 100; i++) {
  const donut = new THREE.Mesh(donutGeometry, material)

  // 隨機位置
  donut.position.x = (Math.random() - 0.5) * 10
  donut.position.y = (Math.random() - 0.5) * 10
  donut.position.z = (Math.random() - 0.5) * 10

  // 隨機旋轉（donut 對稱，半圈即可）
  donut.rotation.x = Math.random() * Math.PI
  donut.rotation.y = Math.random() * Math.PI

  // 隨機大小（三軸必須使用相同值，避免變形）
  const scale = Math.random()
  donut.scale.set(scale, scale, scale)

  scene.add(donut)
}
```

---

## 完整範例

```js
import { FontLoader } from 'three/examples/jsm/loaders/FontLoader.js'
import { TextGeometry } from 'three/examples/jsm/geometries/TextGeometry.js'

// 在 callback 外先載入 texture
const textureLoader = new THREE.TextureLoader()
const matcapTexture = textureLoader.load('/textures/matcaps/1.png')
matcapTexture.colorSpace = THREE.SRGBColorSpace

const fontLoader = new FontLoader()
fontLoader.load('/fonts/helvetiker_regular.typeface.json', (font) => {

  // 文字
  const textGeometry = new TextGeometry('Hello Three.js', {
    font,
    size: 0.5,
    depth: 0.2,
    curveSegments: 12,
    bevelEnabled: true,
    bevelThickness: 0.03,
    bevelSize: 0.02,
    bevelOffset: 0,
    bevelSegments: 5
  })
  textGeometry.center()

  // 共用材質
  const material = new THREE.MeshMatcapMaterial({ matcap: matcapTexture })
  scene.add(new THREE.Mesh(textGeometry, material))

  // 裝飾物件（共用 geometry 和 material）
  const donutGeometry = new THREE.TorusGeometry(0.3, 0.2, 20, 45)
  for (let i = 0; i < 100; i++) {
    const donut = new THREE.Mesh(donutGeometry, material)
    donut.position.set(
      (Math.random() - 0.5) * 10,
      (Math.random() - 0.5) * 10,
      (Math.random() - 0.5) * 10
    )
    donut.rotation.x = Math.random() * Math.PI
    donut.rotation.y = Math.random() * Math.PI
    const s = Math.random()
    donut.scale.set(s, s, s)
    scene.add(donut)
  }
})
```

---

## 常見問題快速排查

| 症狀 | 原因 | 解法 |
|------|------|------|
| 文字偏向某角落，不在中心 | 未置中 | 呼叫 `textGeometry.center()` |
| 文字邊緣模糊或呈鋸齒 | `curveSegments` 太低 | 適當提高（建議 `12`），wireframe 模式確認 |
| fontLoader callback 外的程式碼無效 | 字型非同步載入 | 所有依賴字型的程式碼都必須寫在 callback 內 |
| `FontLoader is not defined` | 從 `THREE` 命名空間引入 | 改從 `three/examples/jsm/loaders/FontLoader.js` 引入 |
| `TextGeometry is not defined` | 同上 | 改從 `three/examples/jsm/geometries/TextGeometry.js` 引入 |
| 大量物件效能低落 | 迴圈內各自建立 geometry / material | 將 geometry 和 material 移到迴圈外共用 |
| donut 比例怪異（拉伸變形） | scale 三軸數值不同 | `const s = Math.random(); donut.scale.set(s, s, s)` |
