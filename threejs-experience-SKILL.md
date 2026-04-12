---
name: threejs-experience
description: >
  當使用者想從零開始建立 Three.js 場景、使用 Experience singleton 架構、
  或將現有 Three.js 程式碼重構為模組化類別結構時，請使用此 skill。
  觸發情境包括：建立新的 Three.js 專案、搭建 Experience / Camera / Renderer /
  World / Sizes / Time / Resources / Debug 等模組、詢問 Three.js 大型專案的
  資料夾結構或最佳實踐。
  只要使用者提到「Three.js singleton」、「Experience 架構」、「Three.js 模組化」，
  就應立即使用此 skill，不需要等使用者明確要求。
---

# Three.js Experience — 從零開始建立指南

所有子模組透過 `new Experience()` 取得唯一實例，不需要手動傳遞參照。

---

## 架構總覽

```
<project-name>/           ← 預設為 web3d，可由使用者指定
├── index.html
├── package.json
└── src/
    ├── main.js
    └── Experience/
        ├── Experience.js        ← Singleton 根節點，掌管一切
        ├── Camera.js
        ├── Renderer.js
        ├── sources.js           ← 資源清單
        ├── World/
        │   ├── World.js         ← 場景物件的容器
        │   └── Environment.js   ← 燈光與環境貼圖（選用）
        └── Utils/
            ├── EventEmitter.js  ← 發布/訂閱基底類別
            ├── Sizes.js         ← 視窗尺寸
            ├── Time.js          ← RAF 時鐘
            ├── Resources.js     ← 資源載入器
            └── Debug.js         ← lil-gui 除錯面板
```

**Experience constructor 內的實例化順序：**
`Debug` → `Sizes` → `Time` → `Scene` → `Resources` → `Camera` → `Renderer` → `World`

---

## STEP 0 — 環境安裝

> **專案名稱**：若使用者有指定名稱則使用之，否則預設為 `web3d`。
> 以下指令中的 `<project-name>` 請替換為實際名稱。

```bash
npm create vite@latest <project-name> -- --template vanilla
# 未指定時：npm create vite@latest web3d -- --template vanilla

cd <project-name>
npm install three
npm install lil-gui
npm run dev
```

**index.html** — 加入 canvas：
```html
<body>
  <canvas class="webgl"></canvas>
  <script type="module" src="/src/main.js"></script>
</body>
```

**src/main.js**：
```js
import Experience from './Experience/Experience.js'
new Experience(document.querySelector('canvas.webgl'))
```

---

## STEP 1 — EventEmitter（發布/訂閱基底）

`Sizes`、`Time`、`Resources` 都繼承此類別。
相較於原版，以下版本以 **Map** 取代 plain object 管理 callbacks，
並用 `WeakRef` 以外的更直覺方式處理 namespace，邏輯更清晰易懂。

```js
// src/Experience/Utils/EventEmitter.js
export default class EventEmitter {
  #listeners = new Map() // Map<eventName, Set<callback>>

  on(event, callback) {
    if (typeof callback !== 'function') {
      console.warn(`[EventEmitter] on('${event}'): callback 必須是 function`)
      return this
    }
    if (!this.#listeners.has(event)) {
      this.#listeners.set(event, new Set())
    }
    this.#listeners.get(event).add(callback)
    return this
  }

  off(event, callback) {
    if (!this.#listeners.has(event)) return this
    if (callback) {
      // 移除特定 callback
      this.#listeners.get(event).delete(callback)
    } else {
      // 移除該 event 的所有 callbacks（用於 destroy）
      this.#listeners.delete(event)
    }
    return this
  }

  trigger(event, ...args) {
    if (!this.#listeners.has(event)) return this
    for (const callback of this.#listeners.get(event)) {
      callback(...args)
    }
    return this
  }
}
```

> **與原版的差異**
> - 使用 `Map` + `Set` 取代巢狀 plain object，查找與移除效率更好
> - 移除 namespace（`.`語法），一般 Three.js 專案用不到此複雜度
> - `off(event)` 不帶 callback 時清除整個 event，適合 `destroy()` 時使用
> - Private field `#listeners` 防止外部意外存取
> - API 不變：`on` / `off` / `trigger`，子類別不需修改

---

## STEP 2 — Sizes（視窗尺寸）

```js
// src/Experience/Utils/Sizes.js
import EventEmitter from './EventEmitter.js'

export default class Sizes extends EventEmitter {
  constructor() {
    super()
    this._onResize = () => {
      this.width = window.innerWidth
      this.height = window.innerHeight
      this.pixelRatio = Math.min(window.devicePixelRatio, 2)
      this.trigger('resize')
    }

    this.width = window.innerWidth
    this.height = window.innerHeight
    this.pixelRatio = Math.min(window.devicePixelRatio, 2)

    window.addEventListener('resize', this._onResize)
  }

  destroy() {
    window.removeEventListener('resize', this._onResize)
  }
}
```

> `_onResize` 存成實例屬性，確保 `removeEventListener` 能正確移除同一個 reference。

---

## STEP 3 — Time（RAF 時鐘）

```js
// src/Experience/Utils/Time.js
import EventEmitter from './EventEmitter.js'

export default class Time extends EventEmitter {
  constructor() {
    super()
    this.start = Date.now()
    this.current = this.start
    this.elapsed = 0
    this.delta = 16        // 預設 ~60fps 的 frame 間隔（毫秒）
    this._rafId = null

    // 用 requestAnimationFrame 延遲第一次 tick，避免 delta = 0
    this._rafId = window.requestAnimationFrame(() => this._tick())
  }

  _tick() {
    const currentTime = Date.now()
    this.delta = currentTime - this.current
    this.current = currentTime
    this.elapsed = this.current - this.start
    this.trigger('tick')
    this._rafId = window.requestAnimationFrame(() => this._tick())
  }

  destroy() {
    if (this._rafId !== null) {
      window.cancelAnimationFrame(this._rafId)
      this._rafId = null
    }
  }
}
```

> `cancelAnimationFrame` 讓 `destroy()` 能真正停止 RAF loop，不需依賴 EventEmitter 的 `off`。

---

## STEP 4 — Debug（除錯面板）

```js
// src/Experience/Utils/Debug.js
import GUI from 'lil-gui'

export default class Debug {
  constructor() {
    // URL 尾端加 #debug 才顯示面板（如 http://localhost:5173/#debug）
    this.active = window.location.hash === '#debug'
    if (this.active) {
      this.ui = new GUI()
    }
  }

  destroy() {
    this.ui?.destroy()
  }
}
```

子模組使用方式：
```js
this.debug = this.experience.debug
if (this.debug.active) {
  this.debugFolder = this.debug.ui.addFolder('myModule')
  this.debugFolder.add(this, 'someValue').min(0).max(1)
}
```

---

## STEP 5 — Resources（資源載入器）

```js
// src/Experience/Utils/Resources.js
import * as THREE from 'three'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import EventEmitter from './EventEmitter.js'

export default class Resources extends EventEmitter {
  constructor(sources) {
    super()
    this.sources = sources
    this.items = {}             // { [name]: 載入完成的資源 }
    this.toLoad = sources.length
    this.loaded = 0

    if (this.toLoad === 0) {
      // 沒有資源時，下一個 tick 再觸發，確保監聽者已註冊完畢
      setTimeout(() => this.trigger('ready'), 0)
      return
    }

    this._setLoaders()
    this._startLoading()
  }

  _setLoaders() {
    this.loaders = {
      gltf:        new GLTFLoader(),
      texture:     new THREE.TextureLoader(),
      cubeTexture: new THREE.CubeTextureLoader(),
    }
  }

  _startLoading() {
    const onLoaded = (source, file) => {
      this.items[source.name] = file
      this.loaded++
      if (this.loaded === this.toLoad) this.trigger('ready')
    }

    for (const source of this.sources) {
      switch (source.type) {
        case 'gltfModel':
          this.loaders.gltf.load(source.path, (f) => onLoaded(source, f))
          break
        case 'texture':
          this.loaders.texture.load(source.path, (f) => onLoaded(source, f))
          break
        case 'cubeTexture':
          this.loaders.cubeTexture.load(source.path, (f) => onLoaded(source, f))
          break
        default:
          console.warn(`[Resources] 未知的資源類型：${source.type}`)
      }
    }
  }
}
```

**資源清單 `src/Experience/sources.js`**：
```js
export default [
  // 取消註解以使用：
  // {
  //   name: 'environmentMapTexture',
  //   type: 'cubeTexture',
  //   path: [
  //     'textures/environmentMap/px.jpg', 'textures/environmentMap/nx.jpg',
  //     'textures/environmentMap/py.jpg', 'textures/environmentMap/ny.jpg',
  //     'textures/environmentMap/pz.jpg', 'textures/environmentMap/nz.jpg',
  //   ]
  // },
  // { name: 'grassColorTexture', type: 'texture',   path: 'textures/dirt/color.jpg' },
  // { name: 'foxModel',          type: 'gltfModel', path: 'models/Fox/glTF/Fox.gltf' },
]
```

---

## STEP 6 — Camera

```js
// src/Experience/Camera.js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import Experience from './Experience.js'

export default class Camera {
  constructor() {
    this.experience = new Experience()
    this.sizes  = this.experience.sizes
    this.scene  = this.experience.scene
    this.canvas = this.experience.canvas

    this._setInstance()
    this._setControls()
  }

  _setInstance() {
    this.instance = new THREE.PerspectiveCamera(
      35,
      this.sizes.width / this.sizes.height,
      0.1,
      100
    )
    this.instance.position.set(6, 4, 8)
    this.scene.add(this.instance)
  }

  _setControls() {
    this.controls = new OrbitControls(this.instance, this.canvas)
    this.controls.enableDamping = true
  }

  resize() {
    this.instance.aspect = this.sizes.width / this.sizes.height
    this.instance.updateProjectionMatrix()
  }

  update() {
    this.controls.update()
  }
}
```

> ⚠️ **context 陷阱**：不可寫 `this.sizes.on('resize', this.resize)`，
> 因為 callback 內的 `this` 會是 `Sizes` 而非 `Camera`。
> 請讓 `Experience` 統一往下呼叫 `camera.resize()`。

---

## STEP 7 — Renderer

```js
// src/Experience/Renderer.js
import * as THREE from 'three'
import Experience from './Experience.js'

export default class Renderer {
  constructor() {
    this.experience = new Experience()
    this.canvas  = this.experience.canvas
    this.sizes   = this.experience.sizes
    this.scene   = this.experience.scene
    this.camera  = this.experience.camera

    this._setInstance()
  }

  _setInstance() {
    this.instance = new THREE.WebGLRenderer({
      canvas: this.canvas,
      antialias: true,
    })
    this.instance.toneMapping = THREE.CineonToneMapping
    this.instance.toneMappingExposure = 1.75
    this.instance.shadowMap.enabled = true
    this.instance.shadowMap.type = THREE.PCFSoftShadowMap
    this.instance.setClearColor('#211d20')
    this.instance.setSize(this.sizes.width, this.sizes.height)
    this.instance.setPixelRatio(this.sizes.pixelRatio) // pixelRatio 已在 Sizes 限制最大值
  }

  resize() {
    this.instance.setSize(this.sizes.width, this.sizes.height)
    this.instance.setPixelRatio(this.sizes.pixelRatio)
  }

  update() {
    // this.camera 是我們的 Camera 類別；THREE.js camera 在 this.camera.instance
    this.instance.render(this.scene, this.camera.instance)
  }
}
```

---

## STEP 8 — World

```js
// src/Experience/World/World.js
import Experience from '../Experience.js'
// import Environment from './Environment.js'

export default class World {
  constructor() {
    this.experience = new Experience()
    this.scene     = this.experience.scene
    this.resources = this.experience.resources

    this.resources.on('ready', () => {
      // ⚠️ 順序很重要：所有 mesh 必須在 Environment 之前建立，
      //    因為 Environment 會遍歷 scene 套用 envMap
      // this.floor = new Floor()
      // this.fox   = new Fox()
      // this.environment = new Environment()
    })
  }

  update() {
    // ⚠️ resources 非同步載入，update() 可能在物件建立前被呼叫
    // if (this.fox) this.fox.update()
  }
}
```

---

## STEP 9 — Experience（Singleton 根節點）

```js
// src/Experience/Experience.js
import * as THREE from 'three'
import Debug     from './Utils/Debug.js'
import Sizes     from './Utils/Sizes.js'
import Time      from './Utils/Time.js'
import Resources from './Utils/Resources.js'
import Camera    from './Camera.js'
import Renderer  from './Renderer.js'
import World     from './World/World.js'
import sources   from './sources.js'

let instance = null

export default class Experience {
  constructor(canvas) {
    // ---- Singleton 守衛 ----
    if (instance) return instance
    instance = this

    // 開發用：在 console 輸入 experience.xxx 快速偵錯
    window.experience = this

    this.canvas = canvas

    // Debug 必須最先實例化，其他模組的 constructor 中可能就需要 debug.active
    this.debug    = new Debug()
    this.sizes    = new Sizes()
    this.time     = new Time()
    this.scene    = new THREE.Scene()
    this.resources = new Resources(sources)
    this.camera   = new Camera()
    this.renderer = new Renderer()
    this.world    = new World()

    // resize 與 tick 由 Experience 統一往下傳播，控制執行順序
    this.sizes.on('resize', () => this.resize())
    this.time.on('tick',    () => this.update())
  }

  resize() {
    this.camera.resize()
    this.renderer.resize()
  }

  update() {
    this.camera.update()
    this.world.update()
    this.renderer.update()  // render call 永遠在最後
  }

  destroy() {
    // 1. 停止 event 監聽
    this.sizes.off('resize')
    this.time.off('tick')

    // 2. 停止 RAF loop 與 window resize listener
    this.time.destroy()
    this.sizes.destroy()

    // 3. 釋放 scene 中所有 geometry / material / texture
    this.scene.traverse((child) => {
      if (child.isMesh) {
        child.geometry.dispose()
        // 同時處理單一 material 與 material array
        const materials = Array.isArray(child.material)
          ? child.material
          : [child.material]
        for (const mat of materials) {
          for (const value of Object.values(mat)) {
            if (value?.isTexture) value.dispose()
          }
          mat.dispose()
        }
      }
    })

    // 4. 釋放 controls 與 renderer
    this.camera.controls.dispose()
    this.renderer.instance.dispose()

    // 5. 釋放 Debug UI
    this.debug.destroy()

    // 6. 重置 singleton，下次可重新建立
    instance = null
  }
}
```

---

## 擴充：新增 Mesh 的固定模式

每個 World 物件都遵循相同結構——透過 `new Experience()` 取得 singleton，
再將設定拆成多個獨立方法：

```js
// src/Experience/World/Floor.js（範例）
import * as THREE from 'three'
import Experience from '../Experience.js'

export default class Floor {
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
    this.geometry = new THREE.CircleGeometry(5, 64)
  }

  _setTextures() {
    this.textures = {}
    this.textures.color = this.resources.items.grassColorTexture
    this.textures.color.colorSpace = THREE.SRGBColorSpace
    this.textures.color.repeat.set(1.5, 1.5)
    this.textures.color.wrapS = THREE.RepeatWrapping
    this.textures.color.wrapT = THREE.RepeatWrapping

    this.textures.normal = this.resources.items.grassNormalTexture
    this.textures.normal.repeat.set(1.5, 1.5)
    this.textures.normal.wrapS = THREE.RepeatWrapping
    this.textures.normal.wrapT = THREE.RepeatWrapping
  }

  _setMaterial() {
    this.material = new THREE.MeshStandardMaterial({
      map: this.textures.color,
      normalMap: this.textures.normal,
    })
  }

  _setMesh() {
    this.mesh = new THREE.Mesh(this.geometry, this.material)
    this.mesh.rotation.x = -Math.PI * 0.5
    this.mesh.receiveShadow = true
    this.scene.add(this.mesh)
  }
}
```

---

## 關鍵設計決策一覽

| 主題 | 說明 |
|---|---|
| **取得 Singleton** | 子模組一律用 `new Experience()`，singleton guard 保證回傳同一實例 |
| **Debug URL hash** | 網址加 `#debug` 才顯示 lil-gui，hash 改變後需手動重整頁面 |
| **resize/update 傳遞** | 永遠從 `Experience` 往下呼叫子模組，父層控制執行順序 |
| **World 中的 guard** | `if (this.fox) this.fox.update()` 防止資源尚未載入就呼叫 update |
| **`setTimeout(() => trigger('ready'), 0)`** | sources 為空時，確保監聽者已在下一個 tick 前完成註冊 |
| **destroy 後重置 instance** | `instance = null` 讓 SPA 路由切換後可重新 `new Experience(canvas)` |
| **後處理注意** | 使用 `EffectComposer` 時，destroy 需額外釋放 composer、RenderTarget 及所有 passes |
