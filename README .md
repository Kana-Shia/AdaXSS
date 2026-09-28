# AdaXSS

**A CRS-Based Adaptive Defense Framework for LLM-Generated XSS Attacks**

AdaXSS 是一套面向 LLM 生成式 XSS 變異攻擊的適應性防禦與評估框架。本專案為國立臺北大學資訊工程學系碩士論文《AdaXSS：面向 LLM 生成式 XSS 攻擊之 CRS 型防禦框架》之實作與實驗程式碼。

> **Author**: Jui-Kuan Liu (劉睿寬) ｜ **Advisor**: Dr. Chin-Yang Tseng (曾俊元) ｜ National Taipei University, 2026

---

## Table of Contents

- [Overview](#overview)
- [Key Contributions](#key-contributions)
- [Architecture](#architecture)
- [Key Results](#key-results)
- [Repository Structure](#repository-structure)
- [Requirements](#requirements)
- [Reproducing the Experiments](#reproducing-the-experiments)
- [Datasets](#datasets)
- [Citation](#citation)
- [License](#license)
- [Disclaimer](#disclaimer)

---

## Overview

OWASP Core Rule Set (CRS) 依賴規則匹配與 anomaly score 門檻：門檻過低易增加誤報，過高則提高漏報。同時，LLM 能快速產生大量語法多樣的 XSS 變異，使機器學習偵測模型面臨分布偏移。

AdaXSS 採用「放寬 CRS 門檻以降低誤報，並以深度學習模型補強增加之漏報」的雙層防禦設計，並持續透過 LLM 生成新型變異樣本更新模型：

| 層級 | 元件 | 職責 |
| --- | --- | --- |
| Layer 1 | ModSecurity + OWASP CRS（以 rule count 為門檻） | 攔截已知攻擊，維持可部署性與可解釋性 |
| Layer 2 | ML-Rescue（TinyBERT + BiLSTM） | 補救通過 CRS 的攻擊（降低漏報） |
| Update | LLM 變異生成 + 結構距離取樣 | 以少量高價值樣本持續微調 ML-Rescue |

## Key Contributions

1. **CRS + ML-Rescue 雙層防禦**：以 rule count 作為 CRS 門檻指標，並以 TinyBERT+BiLSTM 作為 post-CRS 補救層。
2. **CRS Reference 導向之 LLM 變異生成**：將 payload 實際觸發之 CRS 規則與比對條件納入 prompt，並以 Playwright DOM 觸發驗證與 ML Bypass 篩選確保樣本有效。
3. **多輪再變異與結構距離取樣**：以字元層級 TF-IDF 餘弦距離選取結構差異最大之樣本進行模型更新，降低資料冗餘與訓練成本。

## Architecture

```
Stage 1: XSS Payload Generation
  GPT-4o seed generation
    -> Playwright DOM trigger validation + TinyBERT+BiLSTM bypass filter
    -> ModSecurity + CRS (collect triggered rules / CRS Reference)
    -> Claude CRS-guided mutation -> revalidation

Stage 2: Adaptive Defense and Model Update
  Char n-gram TF-IDF structural distance sampling
    -> Farthest 20% : fine-tune ML-Rescue
    -> Remaining 80%: held-out test set
    -> Evaluate CRS Only / CRS + ML-Rescue (thresholds 1-10)
```

| 設定 | 說明 |
| --- | --- |
| Seed 生成 | GPT-4o，經 DOM 觸發與 Baseline 模型繞過篩選，取得 200 筆 seed |
| 變異生成 | Claude，三種設定：No Ref / Local Ref (CRS Audit Log) / Official Ref (CRS Reference) |
| CRS 環境 | `owasp/modsecurity-crs:nginx`，CRS 4.16.0，Paranoia Level 1 |
| CRS Reference 來源 | 依 Rule ID 對應 CRS 4.27.0 官方規則檔 |
| 前端驗證 | Playwright（DOM sink 觸發） |

## Key Results

實驗以 CRS 門檻 3 為基準（Baseline 測試集上 FPR 與 FNR 較為平衡）。

**防禦框架比較（Baseline 測試集，門檻 3）**

| Framework | Accuracy | FPR | FNR | F1 |
| --- | --- | --- | --- | --- |
| CRS Only | 0.8666 | 0.0604 | 0.1960 | 0.8665 |
| CRS + WAFBooster | 0.8569 | 0.1047 | 0.1760 | 0.8612 |
| CRS + CNN+BiLSTM | 0.9704 | 0.0616 | 0.0022 | 0.9732 |
| **CRS + TinyBERT+BiLSTM** | **0.9722** | **0.0604** | **0.0000** | **0.9748** |

**LLM 變異樣本之穿透能力（Joint Success Rate = DOM 觸發且繞過 ML）**

| Generation Method | Evaluated | PL Trigger | ML Bypass | Joint Success |
| --- | --- | --- | --- | --- |
| Claude | 3,913 | 89.78% | 69.15% | 62.05% |
| Claude + CRS Audit Log | 3,914 | 87.38% | 54.98% | 48.44% |
| Claude + CRS Reference | 3,859 | 78.13% | 26.25% | 20.73% |
| LLM-Driven | 3,951 | 18.65% | 25.77% | 9.26% |
| DDQN | 2,077 | 3.66% | 7.56% | 0.43% |

**模型更新效果（Baseline + 80% 變異測試集，門檻 3）**

| ML-Rescue Model | Accuracy | FPR | FNR | F1 |
| --- | --- | --- | --- | --- |
| CRS Only | 0.8695 | 0.0602 | 0.1859 | 0.8746 |
| CRS + ML-Rescue (Baseline model) | 0.9698 | 0.0602 | 0.0066 | 0.9735 |
| CRS + ML-Rescue (Distant model) | 0.9735 | 0.0602 | 0.0000 | 0.9768 |

以結構距離最遠之 20% 樣本微調後，ML-Rescue 在未見 LLM 變異樣本上的 FNR 由 18.59%（CRS Only）降至 0%，且未增加 FPR。結構距離取樣消融實驗中，Distant 策略在相同樣本預算下皆優於 Random 與 Nearest，完整結果請見論文第 4 章。

## Repository Structure

```
.
├── src/
│   ├── XSS_with_TinyBERT_Training.ipynb        # Baseline ML-Rescue training
│   ├── Gpt_XSS_Mutations_Filter.py             # Seed generation and filtering
│   ├── Claude_XSS_Mutations_No_Ref.py
│   ├── Claude_XSS_Mutations_Local_Ref.py
│   ├── Claude_XSS_Mutations_Official_Ref.py
│   ├── Group_All.py                            # PL / ML filtering + CRS audit labeling
│   ├── Coverage_Diversity Comparison.py        # Structural diversity analysis
│   ├── Char_Tsne.py                            # t-SNE visualization
│   ├── XSS_with_TinyBERT_Selete_Strategy.py    # Structural distance sampling ablation
│   ├── XSS_with_TinyBERT_Full_Training.py      # Fine-tuning with Distant 20%
│   ├── Run_Crs_Snapshot.py                     # CRS snapshot labeling
│   └── Run_Crs_Snapshot_Resuce.py              # CRS + ML-Rescue evaluation
├── rule/                                       # CRS rule files (source of CRS Reference)
├── res/
│   ├── train_data/
│   │   ├── xss_dataset.csv                     # Kaggle baseline dataset
│   │   └── successful_set_200.txt              # 200 validated seed payloads
│   └── claude_test_payload/
│       ├── Claude/                             # Claude (no reference)
│       ├── Claude_CRS_Audit_Log/               # Claude + CRS Audit Log
│       └── Claude_CRS_Reference/               # Claude + CRS Reference
├── .env.example                                # API key template
├── requirements.txt
├── LICENSE
└── README.md
```

## Requirements

**Hardware (reference setup)**: Intel Core i9-10850K, 32 GB RAM, NVIDIA GeForce RTX 3060

**Software**

| Package | Version |
| --- | --- |
| Python | 3.9.25 |
| TensorFlow | 2.12.0 |
| Transformers | 4.57.3 |
| PyTorch | 2.8.0 |
| NumPy | 1.23.5 |
| Pandas | 2.3.3 |
| scikit-learn | 1.5.1 |
| Playwright | 1.56.0 |

**External services**

- Docker（執行 [`modsecurity-crs-docker`](https://github.com/coreruleset/modsecurity-crs-docker)）
- OpenAI 與 Anthropic API 金鑰（seed 與變異樣本生成）

```bash
pip install -r requirements.txt
playwright install chromium
```

**API keys**

複製 `.env.example` 為 `.env` 並填入金鑰（`.env` 已列入 `.gitignore`，請勿提交）：

```bash
cp .env.example .env
```

```
OPENAI_API_KEY=your_key_here
ANTHROPIC_API_KEY=your_key_here
```

> 以下所有指令皆須在**專案根目錄**執行，以確保 `res/` 相對路徑正確。

## Reproducing the Experiments

### 0. ModSecurity + CRS 環境設定

於 `modsecurity-crs-docker` 容器中，將 `/etc/modsecurity.d/setup.conf` 設定如下：

```apache
SecRuleEngine On
Include /etc/modsecurity.d/owasp-crs/crs-setup.conf
Include /etc/modsecurity.d/owasp-crs/rules/*.conf

SecAuditEngine On
SecAuditLog /tmp/modsec_audit.log
SecAuditLogParts ABIJDEFHZ
SecAuditLogType Serial
```

實驗固定使用 `PARANOIA=1`，CRS 門檻以 audit log 之 rule count 離線比較。

### 1. 訓練 Baseline ML-Rescue

執行 `src/XSS_with_TinyBERT_Training.ipynb`，以 `res/train_data/xss_dataset.csv`（Kaggle）訓練 TinyBERT+BiLSTM。

- 輸出：`BestModel_TinyBERT_1.keras`

### 2. 建立 Seed Payload

```bash
python src/Gpt_XSS_Mutations_Filter.py
```

篩選可通過 Baseline 模型（ML Bypass）且能觸發 DOM（PL Trigger）之 payload。

- 輸出：`res/train_data/successful_set_200.txt`（200 筆）

### 3. 產生 Claude 變異樣本

```bash
python src/Claude_XSS_Mutations_No_Ref.py        # Claude
python src/Claude_XSS_Mutations_Local_Ref.py     # Claude + CRS Audit Log
python src/Claude_XSS_Mutations_Official_Ref.py  # Claude + CRS Reference
```

| 設定 | 輸出目錄 |
| --- | --- |
| Claude | `res/claude_test_payload/Claude` |
| Claude + CRS Audit Log | `res/claude_test_payload/Claude_CRS_Audit_Log` |
| Claude + CRS Reference | `res/claude_test_payload/Claude_CRS_Reference` |

### 4. 統一驗證（PL、ML、CRS）

完成 [步驟 0](#0-modsecurity--crs-環境設定) 後執行：

```bash
python src/Group_All.py
```

對變異樣本執行 PL Trigger 與 ML Bypass 過濾，並標註 CRS Audit Log 觸發資訊。

### 5. 結構多樣性分析

```bash
python "src/Coverage_Diversity Comparison.py"   # 各生成方法之 cluster coverage 比較
python "src/Char_Tsne.py"                       # Char-TFIDF + t-SNE 二維視覺化
```

### 6. 結構距離取樣消融實驗

```bash
python src/XSS_with_TinyBERT_Selete_Strategy.py
```

比較 Distant / Random / Nearest 三種取樣策略。

### 7. 使用 Distant 20% 微調模型

```bash
python src/XSS_with_TinyBERT_Full_Training.py
```

- 輸出：`BestModel_TinyBERT_1_FT_distant.keras`
- 同時保留 Baseline 加上其餘 80% 變異樣本，作為後續測試資料。

### 8. 建立 CRS Snapshot

```bash
python src/Run_Crs_Snapshot.py
```

對下列兩組資料各執行一次 CRS 標註：

- Kaggle `xss_dataset.csv`
- `xss_dataset.csv` + 其餘 80% 變異樣本

### 9. 評估 CRS + ML-Rescue

```bash
python src/Run_Crs_Snapshot_Resuce.py
```

於 CRS 門檻 1 至 10 評估 Accuracy、FPR、FNR 與 F1-Score。

## Datasets

| Dataset | Description | Size |
| --- | --- | --- |
| [Cross Site Scripting XSS Dataset for Deep Learning](https://www.kaggle.com/) (Kaggle, Syed Saqlain Hussain Shah) | Baseline，來源含 PortSwigger 與 OWASP XSS Cheat Sheet | 13,686（Benign 6,313 / Malicious 7,373） |
| Seed payloads | GPT-4o 生成並通過 DOM 與 ML Bypass 篩選 | 200 |
| LLM variants | Claude 於三種設定下生成，每組約 4,000 筆候選 | see paper |

Baseline 資料集依 9:1 隨機切分為訓練集與測試集。

## Citation

```bibtex
@mastersthesis{liu2026adaxss,
  title  = {AdaXSS: A CRS-Based Framework for LLM-Generated XSS Attacks},
  author = {Liu, Jui-Kuan},
  school = {National Taipei University},
  year   = {2026},
  month  = {July},
  type   = {Master's Thesis}
}
```

## License

The source code in this repository is released under the [MIT License](LICENSE).

The thesis text is © 2026 Jui-Kuan Liu (劉睿寬). All rights reserved
（本論文著作權為劉睿寬所有，並受中華民國著作權法保護）。

## Disclaimer

本專案僅供學術研究與防禦性安全評估使用。專案中之 payload 與生成程式碼不得用於未經授權之系統測試或任何非法用途，使用者須自行承擔相關法律責任。
