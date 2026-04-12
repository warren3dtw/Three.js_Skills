# three-mesh-bvh 使用指南

## 安裝

```bash
npm install three-mesh-bvh
```

---

## 一次性初始化（全局設置）

**在應用啟動時執行一次**，將 BVH 功能掛載到 Three.js 的 prototype：

```javascript
import * as THREE from 'three';
import {
  computeBoundsTree,
  disposeBoundsTree,
  acceleratedRaycast,
} from 'three-mesh-bvh';

// 必須在場景初始化時執行
THREE.BufferGeometry.prototype.computeBoundsTree = computeBoundsTree;
THREE.BufferGeometry.prototype.disposeBoundsTree = disposeBoundsTree;
THREE.Mesh.prototype.raycast = acceleratedRaycast;
```

---

## 建構 BVH

```javascript
// 方法 A：Extension（推薦，最簡單）
geometry.computeBoundsTree();

// 方法 B：手動建構（需要直接控制時）
import { MeshBVH, SAH } from 'three-mesh-bvh';
geometry.boundsTree = new MeshBVH(geometry, {
  strategy: SAH,       // CENTER（預設）/ AVERAGE / SAH
  maxDepth: 40,
  maxLeafSize: 10,
  indirect: false,     // true 保留原始 index 不變
});

// 清理
geometry.disposeBoundsTree();
```

---

## 核心導入清單

```javascript
import {
  // BVH 類別
  MeshBVH,              // 三角網格（主要使用）
  PointsBVH,            // 點雲
  LineBVH,              // Line
  LineSegmentsBVH,      // LineSegments
  SkinnedMeshBVH,       // 骨骼動畫網格
  ObjectBVH,            // 場景整體空間查詢

  // Extension 函式
  computeBoundsTree,
  disposeBoundsTree,
  acceleratedRaycast,
  computeBatchedBoundsTree,   // BatchedMesh 用
  disposeBatchedBoundsTree,

  // 建構策略常數
  CENTER,               // 最快，適合動態更新
  AVERAGE,
  SAH,                  // 最佳質量，適合靜態資源

  // shapecast 回傳值
  NOT_INTERSECTED,
  INTERSECTED,
  CONTAINED,

  // 工具
  BVHHelper,            // 調試可視化
  StaticGeometryGenerator,
  getTriangleHitPointInfo,
  ExtendedTriangle,
  OrientedBox,

  // GPU（路徑追蹤用）
  MeshBVHUniformStruct,
  BVHShaderGLSL,
} from 'three-mesh-bvh';
```

---

## Raycasting

```javascript
// 標準 Three.js raycaster（掛載 acceleratedRaycast 後自動使用 BVH）
const raycaster = new THREE.Raycaster();
raycaster.firstHitOnly = true;             // 只找最近一個，效能更好
raycaster.setFromCamera(mouse, camera);
const hits = raycaster.intersectObjects([mesh]);

// 直接查詢（在 local space 操作）
const bvh = geometry.boundsTree;
const localRay = ray.clone().applyMatrix4(
  new THREE.Matrix4().copy(mesh.matrixWorld).invert()
);
const hit = bvh.raycastFirst(localRay);    // 回傳單一最近結果
const hits = bvh.raycast(localRay);        // 回傳所有命中結果
```

---

## 距離查詢

```javascript
const bvh = geometry.boundsTree;

// 查詢點到網格最近距離
const result = bvh.closestPointToPoint(
  worldPoint,
  {},           // target（可選，接收結果的 Vector3）
  0,            // minThreshold（提前終止距離）
  Infinity      // maxThreshold（最大搜尋距離）
);
// result: { point, distance, faceIndex }

// 取得命中點的詳細資訊（UV、法線等）
import { getTriangleHitPointInfo } from 'three-mesh-bvh';
const info = getTriangleHitPointInfo(geometry, result.faceIndex, result.point);
// info: { normal, uv, uv2, ... }

// 兩個網格之間的最近距離
bvh1.closestPointToGeometry(geometry2, matrixFromGeo2ToGeo1);
```

---

## 碰撞檢測

```javascript
const bvh = geometry.boundsTree;

// 網格 vs Box / Sphere
bvh.intersectsBox(box, meshWorldMatrix);
bvh.intersectsSphere(sphere);

// 網格 vs 網格
bvh.intersectsGeometry(otherGeometry, otherGeomToLocalMatrix);
```

---

## shapecast — 自訂形狀查詢

```javascript
import { INTERSECTED, NOT_INTERSECTED, CONTAINED } from 'three-mesh-bvh';

const sphere = new THREE.Sphere(center, radius);

bvh.shapecast({
  // 必填：判斷節點包圍盒是否與查詢形狀相交
  intersectsBounds: (box, isLeaf, score, depth) => {
    return sphere.intersectsBox(box) ? INTERSECTED : NOT_INTERSECTED;
    // 回傳 CONTAINED 則跳過子節點遍歷，直接處理所有三角形
  },

  // 選填：對每個葉節點中的三角形做精確測試
  intersectsTriangle: (triangle, triIndex, contained) => {
    console.log('命中三角形', triIndex);
    return false; // 回傳 true 則提前停止遍歷
  },

  // 選填：控制節點遍歷順序（回傳分數，越小越先遍歷）
  boundsTraverseOrder: (box) => {
    return box.distanceToPoint(queryPoint);
  },
});
```

---

## bvhcast — 兩個 BVH 互相測試

```javascript
const bvh1 = mesh1.geometry.boundsTree;
const bvh2 = mesh2.geometry.boundsTree;

// 將 mesh2 的 BVH 轉換到 mesh1 的 local space
const matrixToLocal = new THREE.Matrix4()
  .copy(mesh2.matrixWorld)
  .premultiply(new THREE.Matrix4().copy(mesh1.matrixWorld).invert());

bvh1.bvhcast(bvh2, matrixToLocal, {
  intersectsRanges: (offset1, count1, offset2, count2) => {
    // 精確測試三角形對
    return false; // 回傳 true 提前停止
  },
});
```

---

## 多幾何體支援

### BatchedMesh

```javascript
THREE.BatchedMesh.prototype.computeBoundsTree = computeBatchedBoundsTree;
THREE.BatchedMesh.prototype.disposeBoundsTree = disposeBatchedBoundsTree;
THREE.BatchedMesh.prototype.raycast = acceleratedRaycast;

const geomId = batchedMesh.addGeometry(geometry);
batchedMesh.computeBoundsTree(geomId);
// 命中結果包含 hit.batchId
```

### SkinnedMesh（骨骼動畫）

```javascript
import { SkinnedMeshBVH } from 'three-mesh-bvh';

const bvh = new SkinnedMeshBVH(skinnedMesh, { strategy: CENTER });
skinnedMesh.geometry.boundsTree = bvh;

// 動畫更新後同步 BVH
mixer.update(delta);
bvh.refit();
```

### ObjectBVH（場景級查詢）

```javascript
import { ObjectBVH } from 'three-mesh-bvh';

const objectBVH = new ObjectBVH(scene, {
  precise: false,
  includeInstances: true,
});

objectBVH.shapecast({ /* ... */ });
```

---

## refit — 動態幾何體更新

```javascript
// 頂點位置改變後，不需要重建，只需 refit（比重建快得多）
positionAttribute.needsUpdate = true;
geometry.computeVertexNormals();
geometry.boundsTree.refit();
```

---

## 序列化（預計算 BVH）

```javascript
// 序列化（可存入 localStorage / 傳給 Worker）
const bvh = new MeshBVH(geometry);
const serialized = MeshBVH.serialize(bvh, { cloneBuffers: true });

// 反序列化
geometry.boundsTree = MeshBVH.deserialize(serialized, geometry);
```

### Worker 非同步生成

```javascript
import { ParallelMeshBVHWorker } from 'three-mesh-bvh/worker';

const worker = new ParallelMeshBVHWorker();
worker.generate(geometry).then(bvh => {
  geometry.boundsTree = bvh;
});
```

---

## 調試可視化

```javascript
import { BVHHelper } from 'three-mesh-bvh';

const helper = new BVHHelper(mesh);
helper.depth = 15;              // 顯示第幾層節點
scene.add(helper);
helper.update();                // BVH 改變後呼叫

// 清理
helper.dispose();
scene.remove(helper);
```

---

## 效能技巧

| 技巧 | 說明 |
|------|------|
| `raycaster.firstHitOnly = true` | 只需最近命中時大幅提升速度 |
| 靜態資源用 `SAH` | 建構慢，查詢最快 |
| 動態資源用 `CENTER` | 建構快，適合頻繁重建 |
| `geometry.center()` | 幾何體偏移很大時避免浮點誤差 |
| `indirect: true` | 保留原始 index（BatchedMesh 需要） |
| 設定 `near/far` | 縮小 raycaster 搜尋範圍 |

---

## 注意事項

- **直接查詢 BVH 時所有座標必須在 local space**，結果也是 local space，需要手動用 `matrixWorld` 轉換
- BVH 建構時會**修改 `geometry.index`** 的排列順序，如需保留原始 index 請用 `indirect: true`
- **不支援 InterleavedBufferAttribute** 作為 geometry index
- morph targets 和一般 skinning 不會自動更新 BVH，需手動呼叫 `refit()` 或使用 `SkinnedMeshBVH`
- 多個 geometry group 會建立多個 BVH root，group 過多會影響查詢效能
