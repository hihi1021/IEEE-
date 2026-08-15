# GeoHPI-Ensemble

**融合空間可及性與房價指數之多層集成房價預測模型**
*A Multi-layer Ensemble Model for Housing Price Prediction Integrating Spatial Accessibility and Housing Price Index*

> 本專案以臺灣四個都會行政區之實價登錄資料為基礎，結合 POI 空間可及性特徵與房價指數（HPI），建構四層集成式機器學習模型，並進行跨區域驗證。

---

## 目錄

- [研究動機](#研究動機)
- [研究區域](#研究區域)
- [資料來源](#資料來源)
- [特徵工程](#特徵工程)
- [模型架構](#模型架構)
- [實驗結果](#實驗結果)
- [專案結構](#專案結構)
- [環境需求](#環境需求)
- [快速開始](#快速開始)
- [複現說明](#複現說明)
- [已知問題](#已知問題)
- [授權](#授權)

---

## 研究動機

傳統房價預測模型多以建物屬性（面積、屋齡、樓層）為主，較少系統性納入**生活機能可及性**與**時間維度的市場趨勢**。本研究提出三項改進：

1. **空間可及性特徵**：以最近鄰距離量化 POI（捷運、學校、醫療、商業設施）對房價之影響。
2. **時間趨勢校正**：導入房價指數（HPI）作為外生變數，避免時間序列切分造成的分佈偏移。
3. **殘差修正機制**：針對高總價區間預測誤差偏大的問題，設計 ResidualBooster 進行二階段修正。

---

## 研究區域

| 區域 | 樣本數 | 平均總價 | POI 點位 | 軌道運輸 | 狀態 |
|------|--------|----------|----------|----------|------|
| 高雄市左營區 | — | — | — | 捷運紅線 | ✅ 完成 |
| 新北市板橋區 | 3,111 | 約 1,806 萬 | 493 | 捷運／台鐵／高鐵 | 🟡 資料就緒 |
| 新竹縣竹北市 | — | — | — | 台鐵／高鐵（無捷運） | ⬜ 待處理 |
| 桃園市桃園區 | — | — | — | 台鐵桃園站 | ⬜ 待處理 |

> **註**：竹北市無捷運系統，`dist_mrt` 特徵以台鐵／高鐵站距離替代，特徵維度維持一致。

---

## 資料來源

| 資料 | 來源 | 說明 |
|------|------|------|
| 不動產交易 | 內政部不動產成交案件實際資訊資料供應系統 | 篩選住宅類（公寓／華廈／住宅大樓） |
| POI 點位 | OpenStreetMap / 政府開放資料 | 捷運站、學校、醫院、超商、公園等 |
| 房價指數 | 內政部不動產資訊平台 | 季度資料，以交易年月對應 |

---

## 特徵工程

共 **33 個特徵**，分為三類：

**1. 建物屬性**
- 建物移轉總面積、屋齡、樓層／總樓層、房廳衛數
- 主建物佔比 `main_ratio`、車位類型、有無管理組織

**2. 空間可及性（最近鄰距離）**
- `dist_mrt`、`dist_school`、`dist_hospital`、`dist_park`、`dist_convenience` 等
- 以 Haversine 距離計算，單位為公尺

**3. 市場趨勢**
- 房價指數 `hpi`、交易年、交易季

**預處理**
- 目標變數採 `log1p` 轉換，降低右偏影響
- 資料切分：**時間序列切分 70 / 10 / 20**（Train / Val / Test），避免資料洩漏

---

## 模型架構（v6.3）

```
                  ┌─────────────────┐
   Features ─────►│  Ridge (GD)     │──┐
        │         └─────────────────┘  │
        │         ┌─────────────────┐  │   ┌──────────────┐   ┌──────────────────┐
        ├────────►│  XGBoost        │──┼──►│  Stacking    │──►│ ResidualBooster  │──► ŷ
        │         └─────────────────┘  │   │  Meta-learner│   │ (高總價區間修正)  │
        │         ┌─────────────────┐  │   └──────────────┘   └──────────────────┘
        └────────►│  ...            │──┘
                  └─────────────────┘
```

| 層級 | 元件 | 角色 |
|------|------|------|
| L1 | Ridge Regression (Gradient Descent) | 線性基底，捕捉單調關係 |
| L2 | XGBoost | 非線性交互作用與閾值效應 |
| L3 | Stacking Meta-learner | 融合基模型預測 |
| L4 | ResidualBooster | 針對高總價樣本進行殘差修正 |

**ResidualBooster 觸發門檻**

| 區域 | 門檻 |
|------|------|
| 左營 | > 1,500 萬 |
| 板橋 | > 2,000 萬 |

---

## 實驗結果

### 高雄市左營區

| 指標 | 數值 |
|------|------|
| Test R² | **0.8997** |
| SMAPE | **11.30 %** |
| Cross-Validation R² | **0.8923 ± 0.0082** |

### 其他區域

板橋、竹北、桃園實驗結果待補。

---

## 專案結構

```
housing-price-prediction/
├── data/
│   ├── raw/                     # 原始實價登錄 CSV
│   ├── poi/                     # POI 座標資料
│   └── processed/               # 預處理後資料集
├── src/
│   ├── zuoying_housing_v6_3.py  # 左營區完整流程
│   ├── banqiao_v63.py           # 板橋區完整流程
│   ├── banqiao_preprocess.py    # 板橋區預處理
│   └── utils/                   # 共用函式
├── results/
│   ├── figures/                 # 圖表輸出
│   └── metrics/                 # 評估指標
├── docs/
│   ├── progress_report.pdf
│   └── introduction_ieee.pdf
├── requirements.txt
└── README.md
```

---

## 環境需求

```
python >= 3.10
numpy
pandas
scikit-learn
xgboost
matplotlib
seaborn
geopy
```

安裝：

```bash
pip install -r requirements.txt
```

---

## 快速開始

```bash
# 1. 板橋區資料預處理
python src/banqiao_preprocess.py

# 2. 訓練與評估
python src/banqiao_v63.py

# 3. 結果輸出於 results/
```

---

## 複現說明

- 隨機種子固定為 `RANDOM_STATE = 42`
- 時間序列切分依交易年月排序，**不使用隨機切分**
- 評估指標：R²、SMAPE、MAE、RMSE（皆於還原尺度後計算）

---

## 已知問題

- **`main_ratio` 混合格式**：板橋區原始資料中約 978 筆之主建物佔比以百分比形式儲存（如 `65.3`），其餘為小數（如 `0.653`）。`banqiao_preprocess.py` 已加入自動偵測與校正邏輯。
- 竹北市無捷運系統，特徵定義需另行對齊，跨區比較時須註明。

---

## 引用

```bibtex
@inproceedings{geohpi2025,
  title     = {A Multi-layer Ensemble Model for Housing Price Prediction
               Integrating Spatial Accessibility and Housing Price Index},
  author    = {Aaron},
  booktitle = {2025 International Automatic Control Conference (Automation)},
  year      = {2025},
  address   = {Hsinchu, Taiwan}
}
```

---

## 授權

本專案僅供學術研究使用。實價登錄與 POI 資料之使用須遵循原始資料提供者之授權條款。
