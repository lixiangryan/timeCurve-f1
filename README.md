# TimeCurve F1 Implementation

本專案旨在透過 Time Curves 技術視覺化 F1 賽車手的駕駛風格異同，目前專注於 2024 日本大獎賽 (Max Verstappen, Sergio Perez, Yuki Tsunoda)。

## 📁 專案結構

```
timecurve_f1/
```

## 🛠️ 資料接口使用方式 (Data Collection)

我們使用 [OpenF1 API](https://openf1.org/) 獲取比賽數據。相關代碼位於 `utils/fetch_f1_data.py`。

### 執行方式
```bash
python utils/fetch_f1_data.py
```

### 實作細節
1. **Meeting & Session**: 自動搜尋 2024 日本站 (Japan) 的正賽 (Race) Session Key。
2. **Chunked Fetching**: 由於 `car_data` 和 `location` 資料量大，API 容易回傳 422 或 500 錯誤，因此程式採用 **「每分鐘為一區塊 (Time Chunks)」** 的方式下載。
3. **Data Merging**: 下載後會自動去除重複的時間點 (基於 `date` 欄位)，並存入 CSV。
4. **Retry Logic**: 針對 500 錯誤進行簡單的錯誤記錄與重試機制（目前主要依賴縮小時間窗口來規避）。

---

## 📈 Time Curves 實作流程

本專案將依照以下學術定義的標準流程來實作 Time Curves：

### 1. 資料抽象化 (Data Abstraction)
將比賽數據定義為時序資料集 $P = \{p_0, p_1, ..., p_n\}$。
*   **時間戳記 ($t_i$)**: 對應遙測數據的每個採樣時間點 (約 3-4Hz)。
*   **資料快照 ($s_i$)**: 設定為高維特徵向量，目前簡化為 `[throttle, brake, n_gear]` 以聚焦於駕駛操作。

### 2. 建構距離矩陣 (Distance Matrix Construction)
計算所有時間點兩兩之間的相似度，生成 $N \times N$ 對稱矩陣 $D$。
*   **Metric**: 針對數值型遙測數據，計畫使用 **歐幾里得距離 (Euclidean Distance)** 或 **餘弦相似度 (Cosine Similarity)**（需先正規化）。

### 3. 降維投影 (Dimensionality Reduction)
將高維特徵轉化為 2D 平面座標。
*   **演算法**: 使用 **Classical MDS (Multidimensional Scaling)**。
*   **理由**: 收斂速度快且定位比 Force-directed 方法更精確，適合保留全域的時間結構。

### 4. 幾何優化與調整 (Geometric Refinement)
提升視覺可讀性 (Legibility) 的後處理步驟：
*   **移除重疊 (Overlap Removal)**: 使用迭代演算法推開重疊的點，並為被移動的點加上灰色光暈 (Halo)。
*   **旋轉 (Rotation)**: 自動旋轉圖形，使起始點 $p_0$ 位於左側，整體時間流向往右發展，符合閱讀習慣。

### 5. 曲線繪製 (Curve Drawing)
將散點連接成流暢的曲線。
*   **方法**: 使用 **Catmull-Rom 插值** 或 **貝茲曲線 (Bézier curves)**。
*   **參數**: 平滑參數 $\sigma = 0.3$，並在切線上加入微小隨機擾動，避免來回震盪的路徑完全遮擋。


### 6. 視覺編碼 (Visual Encoding)
將時間與狀態資訊編碼為視覺屬性：
*   **顏色 (Color)**: 代表時間 $t$ 的進程（例如：淺色 $\rightarrow$ 深色），使用 Viridis 色票，紫色為起始，黃色為結束。
*   **粗細 (Thickness)**: 代表「持續時間 (Duration)」。兩點間隔時間越長（速度慢），線條越粗。
*   **光暈 (Halo)**: 
    *   🔵 藍色: 完全相同的重複狀態。
    *   ⚪ 灰色: 被演算法推開的相似群聚。

---

## 📊 圖表解讀與分析結果

### 1. 全局 Time Curve (`output/time_curve_*.png`)
這張圖展示了車手在整個比賽或完整單圈中的「狀態演變」。
- **迴圈 (Loops)**: 代表車輛經歷了一系列相似的狀態後回到了原點。在 F1 中，**每個迴圈通常代表一圈 (Lap)**。
- **軌跡平滑度**:
    - **平滑**: 代表駕駛節奏穩定，每一圈的煞車、加速點都非常一致（例如 Max Verstappen）。
    - **雜亂/毛刺**: 代表每一圈的處理方式都有微小差異，可能受輪胎衰退或車流影響。

### 2. S 彎道比較分析 (`output/comparison_scurve.png`)
我們將三位車手 (Max, Perez, Tsunoda) 在鈴鹿賽道 S 彎 (Sector 1) 的數據投影到同一個相似度空間中。
- **重疊**: 若兩條線緊密重疊，代表兩位車手在該路段的駕駛方式（速度、檔位、油門控制）幾乎完全雷同。
- **分歧 (Divergence)**: 若線條分開，代表駕駛風格出現差異。

#### 🏁 關鍵發現：Max vs Perez (S-Curves)
根據我們的演算法分析 (詳見 `output/divergence_report.txt`)，兩者在入彎處出現了顯著分歧：
*   **Max Verstappen (Index 38)**: 採取了 **激進減速 (V-Style)**。
    *   速度: **209 km/h** | 煞車: **100% (重踩)** | 油門: **0% (全放)**
    *   *解讀*: Max 傾向於晚煞車並重煞，利用重心轉移快速讓車頭對準出彎點。
*   **Sergio Perez (Index 38)**: 保持 **高速滑行 (U-Style)**。
    *   速度: **225 km/h** | 煞車: **0%** | 油門: **94% (幾近全油)**
    *   *解讀*: 在同一時刻，Perez 尚未開始重煞或選擇以更高底速過彎，這顯示了兩者在車輛調校或駕駛習慣上的根本差異。

### 3. 車手一致性分析 (Driver Consistency - `output/consistency_Max.html`)
我們深入分析了 **Max Verstappen** 在 53 圈比賽中，每一次通過 S 彎的操作重疊圖。
- **背景 (Ghost Lines)**：顯示了整場比賽的所有軌跡。較窄的通道代表極高的穩定性。
- **演變 (Evolution)**：
    - 通過觀察顏色從紫色（早期）漸變到黃色（晚期），可以發現隨著輪胎耗損，操作軌跡是否發生偏移。
    - 離群值 (Outliers) 通常對應到被套圈車阻擋或失誤的單圈。

### 4. 互動式視覺化 (`output/index.html` & `output/consistency_Max.html`)
靜態圖表難以呈現時間差。請開啟此網頁文件：
- **[主比較] index.html**: 三位車手 (Max, Perez, Tsunoda) 的動態追逐。
- **[新功能] consistency_Max.html**: 單一車手 (Max) 的 53 圈演變動畫。使用滑桿查看每一圈的差異。
- 網頁內均包含詳細的 **「圖表解讀指南 (How to Read)」**。
- **注意 (Known Observation)**：目前的資料點之間的時間間隔並非固定，約在 **0.4秒 ~ 1.3秒** 之間浮動。這是由於資料來源的不規則性與合併策略所致，目前保留此特性以忠實呈現原始資料的分佈狀況。
- 透過拖動時間軸，您可以精確重現上述「Max 重煞 vs Perez 衝刺」的關鍵瞬間。
- **圖表解讀指南**: 網頁下方已新增詳細說明，解釋相似度空間 (Similiarity Space) 的 X/Y 軸意義與觀察重點。

---

## 🚀 下一步計畫
1. **(已完成)** `scripts/generate_time_curve.py` 基礎版本，可讀取 `data/` 中的 CSV 並計算 MDS。
2. **(已完成)** `scripts/analyze_corner.py` 完成 S 彎切片與多車手比較。
3. **(已完成)** `scripts/interpret_divergence.py` 自動化尋找駕駛風格差異點。
4. **(已完成)** `output/index.html` 實作 D3.js 互動式動畫。
