# Prompt：3D Tutting 模擬器（Mixamo 骨架 + 精準 FK 面板）

請幫我用**原生 ES Module + Three.js**做一個單一 HTML 檔案的 3D Tutting（機械舞手勢）姿勢模擬器，並內建一個能精準控制每一節骨骼 X/Y/Z 角度的 FK 面板。

## 技術規範
- 不使用 build 工具，純瀏覽器可直接開啟的單一 `.html` 檔
- 用 `<script type="importmap">` 設定：
  - `"three"` → `https://unpkg.com/three@0.160.0/build/three.module.js`
  - `"three/addons/"` → `https://unpkg.com/three@0.160.0/examples/jsm/`
- 主邏輯用 `<script type="module">`，以標準 `import` 載入 `GLTFLoader` 與 `OrbitControls`
- 角色模型固定載入：`https://threejs.org/examples/models/gltf/Xbot.glb`（內建 Mixamo 骨架）

## 模型處理
- 用 `GLTFLoader` 非同步載入模型，載入完成前顯示 loading 畫面，失敗時顯示錯誤訊息並在 console 印出
- 載入後：
  - 計算 `Box3` 包圍盒，將模型置中（x/z 置中、y 貼地）
  - 依模型高度等比縮放到固定身高（例如 1.75 單位）
  - 加入地板網格（`GridHelper`）
- 用 `traverse()` + 名稱後綴比對（`isBone && name.endsWith(suffix)`）找出以下 19 根 Mixamo 骨骼，找不到就跳過該骨骼、不要讓程式當掉：
  ```
  Hips, Spine, Spine1, Spine2, Neck, Head,
  RightShoulder, RightArm, RightForeArm, RightHand,
  LeftShoulder, LeftArm, LeftForeArm, LeftHand,
  RightUpLeg, RightLeg, RightFoot,
  LeftUpLeg, LeftLeg, LeftFoot
  ```

## 姿勢疊加機制（重點）
- 載入完成當下，記錄上述每根骨骼當前的 `quaternion` 作為 **rest pose**
- 之後所有姿勢一律「相對 rest pose 疊加」，公式：
  `bone.quaternion = restQuat * Quaternion.fromEuler(currentDeltaXYZ, "XYZ")`
  絕對不要直接覆寫骨骼旋轉，否則會破壞模型原本的綁定姿勢
- 每幀用線性插值（lerp，係數約 0.28）讓「目前角度」平滑趨近「目標角度」

## 預設姿勢
- 內建至少 8 組預設 tutting 姿勢（待機、方框、King Tut、機械人、爆炸、凍結、斜切、十字），每組只需指定用到的關節（雙手上臂/前臂/手掌、頭、脊椎），沒指定到的關節（例如肩胛、腿部、頸部、脊椎中/上段）維持使用者當下手動調整的數值，不要被預設姿勢覆蓋
- 提供「隨機姿勢」：從離散角度候選（如 -180/-135/-90/-45/0/45/90/135/180 的倍數）隨機組合手臂姿勢，模擬即興感
- 提供「自動播放」：可調 BPM（60–220），依節奏定時切換到下一個預設姿勢，並有機率插入隨機姿勢

## 精準 FK 微調面板（核心需求）
- 面板要涵蓋前述全部 19 根骨骼，且**每根骨骼都要有完整 X/Y/Z 三軸**可調（不是只給部分軸）
- 用可摺疊區塊（例如 `<details><summary>`）依身體部位分組：軀幹/頭部、右臂、左臂、右腿、左腿，避免一次攤開近 60 個滑桿造成視覺負擔
- 每一軸一列，包含：
  - 拖曳式 range 滑桿（範圍 -180°~180°，step 0.5，方便快速調整）
  - 數字輸入框，與滑桿雙向同步，可直接打字輸入精確角度
  - 單軸「歸零」小按鈕，快速把這一軸重置為 0
- 找不到對應骨骼的關節，該分組不要顯示對應列
- 切換預設姿勢、按重置、或套用 JSON 後，面板上所有滑桿與數字框要同步更新成目前的目標角度

## 姿勢 JSON 匯出/匯入
- 提供一個可切換顯示的面板，內含一個 `<textarea>`：
  - 開啟時自動填入目前所有關節角度的 JSON（結構為 `{ 關節key: [x,y,z], ... }`，數值四捨五入到小數點下一位）
  - 「套用 JSON」按鈕：解析 textarea 內容，把符合格式的關節角度寫回目標姿勢，格式錯誤要跳出提示而不是讓程式壞掉
  - 「複製」按鈕：選取 textarea 內容並複製到剪貼簿，方便使用者把調好的姿勢存到別處，之後可以貼回來或做成新的預設姿勢

## 相機與互動
- 用 `OrbitControls`：拖曳旋轉視角、滾輪縮放（設定 min/max 距離），初始視角要能完整看到角色全身，`target` 對準角色中段

## 視覺風格
- 深色科技感背景（近黑帶藍紫），搭配 `Fog` 霧化
- 環境光 + 白色主光源 + 洋紅/粉色邊緣光，營造舞台聚光氛圍
- 底部 UI 半透明漸層黑底，含播放/停止、BPM 滑桿、隨機姿勢、重置全部、姿勢 JSON 開關、一排姿勢按鈕（選中要高亮），下方接精準 FK 面板
- 左上角標題文字、右上角簡短操作提示
- 全部文字使用繁體中文
