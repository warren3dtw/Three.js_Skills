# Three.js Skills

本 repo 收錄一系列專為 Three.js 開發設計的 **Claude Code Skills**，讓 Claude 在協助你開發 Three.js 專案時，能依據場景自動載入對應的知識與實作準則。

---

## 什麼是 Skill？

Skill 是 Claude Code 的可擴充知識模組。一個 `.skill` 檔案本質上是打包好的 `SKILL.md`，裡面寫給 Claude 看的是：

- 在什麼情況下應該觸發這個 skill（`description` 欄位）
- 面對該主題時，應遵循哪些架構決策與實作步驟

Skill 不是給人類讀的文件，而是讓 Claude 在對話中能夠「知道正確做法」的知識載體。

---

## Skill 如何被觸發

### 明確觸發（斜線指令）

在 Claude Code 的對話框輸入斜線指令，Claude 會立即載入對應 skill：

```
/threejs-camera
/threejs-materials
/threejs-singleton-scene
```

### 自動觸發（自然語言）

Claude 會根據每個 skill 的 `description` 欄位，判斷當前對話是否符合觸發條件，自動載入對應的 skill，不需要手動呼叫。

> 例如：你說「幫我加一個可以用滑鼠旋轉場景的相機」，Claude 會自動載入 `threejs-camera` skill 並遵照其實作準則回答。

**`description` 的品質決定自動觸發的準確度。** 好的 description 應列舉所有可能的觸發情境，包括使用者不一定會說出「相機」但實際上是在問相機問題的情況。

---

## 安裝方式

### 全域安裝（所有專案共用）

將 `.skill` 檔案複製到 Claude Code 的全域 skills 資料夾：

**Windows：**
```
%APPDATA%\Claude\skills\
```

**macOS / Linux：**
```
~/.claude/skills/
```

### 專案安裝（只在當前專案有效）

將 `.skill` 檔案放到專案內的 `.claude/skills/` 資料夾：

```
your-project/
└── .claude/
    └── skills/
        ├── threejs-camera.skill
        ├── threejs-materials.skill
        └── threejs-singleton-scene.skill
```

---

## 如何撰寫自己的 SKILL.md

一個 skill 由一個 `SKILL.md` 組成，格式如下：

```markdown
---
name: your-skill-name
description: >
  觸發條件的完整描述。這段文字是給 Claude 看的，
  用來判斷當前對話是否需要載入這個 skill。
  應涵蓋所有可能的觸發關鍵字與情境，
  包括使用者不一定會用精確詞彙描述問題的情況。
---

# Skill 標題

這裡是 skill 的主體內容，完全給 Claude 看。
可以包含：
- 架構決策與背景
- 分步驟的實作流程
- 程式碼範例
- 常見錯誤與排查方式
```

### description 撰寫技巧

| 好的 description | 不好的 description |
|-----------------|-------------------|
| 涵蓋多種觸發詞彙（相機、鏡頭、視角、OrbitControls） | 只寫「Three.js 相機」 |
| 說明即使使用者沒用精確詞也應觸發 | 過於簡短，Claude 難以判斷觸發時機 |
| 列舉具體操作情境（Z-fighting、resize、滑鼠控制） | 僅描述技術名詞 |

---

## 如何打包成 `.skill` 檔案

`.skill` 檔案是一個 zip 壓縮檔，內含 `SKILL.md`，壓縮時需保留目錄結構：

```
skill-name/
└── SKILL.md
```

**打包指令（macOS / Linux）：**
```bash
zip threejs-camera.skill threejs-camera/SKILL.md
```

**打包指令（Windows PowerShell）：**
```powershell
Compress-Archive -Path threejs-camera\SKILL.md -DestinationPath threejs-camera.skill
```

打包完成後，將 `.skill` 檔案放到對應的 skills 資料夾即可生效。

---

## 本 Repo 的 Skills

| Skill 檔案 | 觸發指令 | 涵蓋範圍 | 詳細文件 |
|-----------|---------|---------|---------|
| `threejs-experience.skill` | `/threejs-experience` | Experience Singleton 架構、EventEmitter、Sizes、Time、Resources、Renderer | [SKILL.md](threejs-experience-SKILL.md) |
| `threejs-camera.skill` | `/threejs-camera` | PerspectiveCamera、OrthographicCamera、OrbitControls、滑鼠控制 | [SKILL.md](threejs-camera-SKILL.md) |
| `threejs-materials.skill` | `/threejs-materials` | 所有內建材質、PBR、貼圖設定、環境貼圖、lil-gui Debug | [SKILL.md](threejs-materials-SKILL.md) |
| `threejs-raycaster.skill` | `/threejs-raycaster` | 光線投射、滑鼠 hover、物件點擊事件、GLTF 模型互動、動畫碰撞偵測 | [SKILL.md](threejs-raycaster-SKILL.md) |

> 各 skill 的詳細實作內容請參閱對應的 `*-SKILL.md` 檔案。

---

## 使用範例

**你想在場景中加入可旋轉的相機：**
```
幫我在 Three.js 場景中加入 OrbitControls，並設定阻尼效果
```
Claude 自動觸發 `threejs-camera` skill，遵照架構準則實作。

**你想了解 PBR 材質的設定：**
```
/threejs-materials
幫我設定一個有 normalMap 和 roughnessMap 的 MeshStandardMaterial
```
明確觸發 skill 後，Claude 依照 skill 內的步驟指引回答。

**你想從零開始建立一個 Three.js 專案：**
```
/threejs-experience
幫我建立一個新的 Three.js 專案，專案名稱是 my-portfolio
```
