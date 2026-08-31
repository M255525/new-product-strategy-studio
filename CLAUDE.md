# CLAUDE.md

本檔案為 Claude Code 在此子資料夾工作時的指引。此資料夾**本身是獨立 git 儲存庫**，不受根目錄工作區規則約束（除語言等全域偏好）。

## 這是什麼

新品開發策略工作室——單檔前端工具：填一次「產品基本資料」（產品名稱／產品類別／目標客群／市場痛點或需求），四個分頁分別產出**產品五層次分析**（運用行銷學經典的「產品五層次」理論框架：核心利益／基本產品／期望產品／附加產品／潛在產品——**文案刻意不冒充理論創始人本人第一人稱**，只說「運用行銷學經典的『產品五層次』理論框架」，AI prompt 也把 AI 定位為「熟悉相關理論的產品策略顧問」）、**SWOT 分析**、**定價策略建議**、**競品比較**。型態仿 `business-idea-generator`（規則式為主＋選用AI優化＋序號授權鎖整個工具），但這裡是 4 個模組共用一組輸入，而非單一功能。

## 架構

- `index.html` — 表單、4 分頁 UI、授權遮罩、跑馬燈、PWA安裝、訪客徽章、列印匯出，全部內嵌單一 IIFE `<script>`（另有 3 個獨立 IIFE：跑馬燈／PWA安裝／授權閘門，互不相依）。
- `data.js` — `window.NPS_DATA`：`CATEGORY_KEYWORDS`（六個產業桶的分類關鍵字）、`classifyCategory()`、四個模組各自的模板庫（`FIVE_LEVEL_TEMPLATES`／`SWOT_TEMPLATES`／`PRICING_TEMPLATES`＋`PRICING_STRATEGY_PREFERENCE`／`COMPETITOR_TEMPLATES`）、`PRESETS`（5組範例）。**這是規則引擎的唯一真實來源**，調整任一模組的文案措辭只需要改這個檔案，不用動 `index.html` 的邏輯。

### 規則式引擎（免API金鑰，同輸入必同輸出）

`classifyCategory(category, pain)` 用關鍵字比對把使用者填的「產品類別＋痛點」文字分類到 `healthTech/foodBeverage/saasSoftware/homeAppliance/eduContent/general` 六桶之一。沿用 `business-idea-generator` 的 `hashString()`（djb2變體字串hash）取代 `Math.random()` 作確定性選模板：

- `generateRuleFive()` — 五層次每層從對應桶的模板陣列（2變體）用 hash 挑一則，套入 `{{PRODUCT}}/{{TARGET}}/{{PAIN}}`。
- `generateRuleSwot()` — SWOT 四象限各桶固定3則模板（不需hash挑選，全部套用）。
- `generateRulePricing()` — 若使用者填了「成本」與「市場參考價」，前端公式算出建議價格區間（`cost*1.3~cost*1.8`，並依區間與參考價的相對位置＋定位選填值決定策略類型：滲透/撇脂/價值/競爭定價）；若欄位不全，改用 `PRICING_STRATEGY_PREFERENCE[category]` 前2順位用 hash 挑一個。**建議價格區間永遠是前端算出來的，不信任AI回傳的任何數字。**
- `generateRuleCompetitor()` — 比較表格直接用使用者填的 `competitors[]` 資料組成（不虛構競品），只有「差異化建議/風險」段落用 hash 從對應桶模板挑一則。

### AI優化路徑（選用，BYOK）

`AI_PROVIDERS`／`callLLM()`／`extractJsonObject()` 與 `business-idea-generator/index.html` 同一套實作（Claude 需 `anthropic-dangerous-direct-browser-access` header；OpenAI/Gemini/OpenRouter 無此限制；429/500/503/529 重試3次；180秒逾時）。四個模組各自一組 `buildXxxPrompt()`／`validateXxxAi(parsed, ruleResult)`，逐欄位驗證，缺漏或無效的欄位個別退回規則式結果（`missing[]`＋`textSource:'ai'|'mixed'|'rule'`），不整批放棄——比照 `business-idea-generator` 的 `validateAiIdea()` 精神。四個分頁**共用同一組** provider/model/apiKey 設定面板，不是各自獨立設定；`runGeneration()` 是四個分頁共用的產生流程封裝（沒填金鑰直接用規則式結果、有填金鑰才呼叫AI並在失敗時自動退回規則式）。

### localStorage

`npsDraft`（`{productName,category,target,pain,cost,refPrice,positioning,competitors[],results:{five,swot,pricing,competitor}}`，四分頁共用一個 key，切換分頁不遺失其他分頁已產生的內容）、`npsApiConfig`（`{provider,model,apiKey}`）、`npsMarquee`（跑馬燈快取）、`npsSerial`（序號授權）。

## 序號授權（鎖定整個工具，12 個月）

比照 `ai-video-prompt-studio`／`business-idea-generator` 的「單一工具、整個鎖住」做法：`#licenseGate` 全螢幕遮罩預設鎖定，驗證通過才加上 `.hidden`；載入時一律對後端即時重驗（不只信任 localStorage 快取），背景每 20 分鐘重驗一次。

- `Code.gs` — 部署到 Google Sheet 的 Apps Script 原始碼，與 `business-idea-generator/Code.gs` 邏輯完全相同（`doPost` 只做序號驗證＋首次自動啟用，`VALID_AMOUNT=12`）。
- **綁定的 Google Sheet**：<https://docs.google.com/spreadsheets/d/1uD0pPmhoh-jdBe0wHdYJr6p0LlwKWhpPfw0Me1IYBHo/edit>——**使用者指定沿用的任務追蹤表**（表頭「任務／優先順序／負責人／狀態／序號／開始日期／結束日期／交件／附註」，與 `business-idea-generator` 綁定的表結構一致但是不同份，測試序號同樣是 `mark0131`）。過程：第一次嘗試用 Google Drive MCP 直接建立新 Sheet 被 auto-mode 分類器擋下，取得使用者明確授權後改用 clasp 建了一份專屬 Sheet 並部署完成；使用者隨後傳來這份既有任務追蹤表的連結（初版表頭是「里程碑」沒有「序號」欄，使用者補上「序號」欄＋填入 `mark0131` 後請 Claude 確認），於是改綁到這份 Sheet、原本自建的那份已用 `trash_file` 清掉。**部署踩坑**：`clasp create --type sheets --parentId <id>` 併用 `--type` 時 API 會忽略 parentId 綁定既有檔案的語意、直接新建一份空白 Sheet；必須 `clasp create --parentId <id>`（**不加 `--type`**）才能正確綁定到既有 Sheet——之後其他工具要綁定既有 Sheet 部署時務必記得省略 `--type`。**部署方式**：`clasp create --parentId <SheetID>` → 複製 `Code.gs` 內容 → `appsscript.json` 加 `webapp:{executeAs:"USER_DEPLOYING",access:"ANYONE_ANONYMOUS"}` → `clasp push --force` → `clasp deploy`，全程跳過瀏覽器複製貼上與部署精靈。`LICENSE_CHECK_URL` 已回填：`https://script.google.com/macros/s/AKfycbzvPXvkxcG2XHqa0lkBBx_C00q0tmmq9bru0JqdNEye4gVh6UHv1ohSX5gaWqz60ySz/exec`。**使用者已完成一次性 OAuth 授權（2026-08-31）**：`clasp deploy` 用 API 建立部署本會跳過瀏覽器部署精靈附帶觸發的授權（需使用者到 Apps Script 編輯器手動執行一次 `doGet` 完成，這步驟涉及 Google 帳號互動同意畫面，Claude 無法代勞），使用者完成後已用 Playwright 對正式上線網址測試序號 `mark0131`，正確顯示「✓ 剩餘487天可用」並解鎖，**序號授權已完整可用**。clasp 部署用的暫存資料夾 `.gas-deploy/` 已加入 `.gitignore`，不進版控。
- **這支後端只做序號驗證，不代理任何付費 API**，也**不處理跑馬燈**（跑馬燈沿用工作區既有共用端點，見下）。

## 頂部共用跑馬燈

`#marqueeBar` 內容抓自工作區既有的共用授權伺服器（`https://script.google.com/macros/s/AKfycbwKX0.../exec`，與 `business-idea-generator`／`coffee-ig-planner`／`Prompt` 等多個姊妹工具共用同一個 Google Sheet），做法完全比照 `business-idea-generator/index.html`。改跑馬燈內容直接編輯共用 Sheet 即可，不需要重新部署任何 Apps Script。

## PDF 匯出浮水印（2026-08-31 新增）

`#pdfWatermark`（`<img id="wmImg">`）比照 `restaurant-feasibility-calculator`／`business-idea-generator` 的做法：內嵌 base64 data URI（不是 CSS `background-image`，會被瀏覽器「列印背景圖形」選項預設擋掉，`<img>` 是內容一定會印出來）、`position:fixed`＋`@media print`下 flex 置中、`opacity:.11`、`width:42%`（`min/max-width` 200-380px 夾住），讓每一頁都重複出現。**圖檔是使用者本次直接提供的新版「馬克老師 AI・工具・學習・成長」圖示**（非沿用 `IPA_Kano/watermark-source.png` 那份舊版——原圖 1536×1024、已去背透明background，裁掉透明邊界後縮到寬 480px 存成本專案根目錄 `watermark-source.png`，202KB），跟其他專案共用的舊版浮水印不是同一個檔案。base64 字串（約270KB）透過 Python 腳本直接字串替換塞進 `index.html` 的 `WATERMARK_DATA_URI` 常數，沒有經過對話視窗。`init()` 載入時把 `WATERMARK_DATA_URI` 指派給 `#wmImg` 的 `src`。已用 Playwright 驗證：`#wmImg.src` 正確載入 base64 圖片、按「匯出 PDF」後 `buildPrintReport()`＋`window.print()` 正確觸發。

## 隱私與警語

無伺服器端經手使用者資料（序號授權後端除外，只傳送序號本身）；產品資料、競品資訊、AI設定皆只存在使用者瀏覽器的 localStorage。首頁與手冊皆明列使用警語：生成內容僅供新品開發發想參考不構成商業決策建議、AI內容需自行查核、請勿輸入真實個資或機密資料、僅供教學與個人使用禁止商業化。

## 部署狀態：已全部完成（2026-08-31）

本機、GitHub repo（<https://github.com/M255525/new-product-strategy-studio>）、GitHub Pages（<https://m255525.github.io/new-product-strategy-studio/>，Actions workflow 部署）、序號授權後端皆已就緒並端對端驗證通過。

## 本次未做（後續視需要再處理）

- 未打包可攜式桌面版 exe（需求未明確提及）。

## 指令

無建置/測試指令。修改 `index.html`／`data.js`／`manual.html` 後直接用瀏覽器開啟驗證，或暫起 `python -m http.server 8803 --directory 行銷內容工具/new-product-strategy-studio` 測完關閉。修改內嵌 `<script>` 後可用以下方式快速檢查語法：

```bash
python -c "
import re
html = open('index.html', encoding='utf-8').read()
open('_check.js','w',encoding='utf-8').write(re.findall(r'<script>(.*?)</script>', html, re.S)[0])
"
node --check _check.js
node --check data.js
```

**測試序號授權邏輯前，需先照 `SETUP-授權伺服器設定.md` 部署好 Apps Script 並回填 `LICENSE_CHECK_URL`**，否則會顯示「尚未設定授權伺服器網址」的 fail-closed 錯誤訊息並停留在鎖定畫面；開發階段要測試四個模組的產生邏輯，可在瀏覽器 devtools 手動對 `#licenseGate` 加上 `hidden` class 暫時繞過。

驗證 AI 路徑不需要真實金鑰：可在瀏覽器 console 攔截 `window.fetch` 回傳假的 provider 回應格式，確認 `callLLM → extractJsonObject → validateXxxAi → renderXxx` 整條管線正確（含單一欄位驗證失敗時的單點 fallback），測完記得還原 `window.fetch`。已用 Playwright 端對端驗證過：5組範例在四個模組皆能零金鑰產生完整內容、分頁切換、競品動態列新增/刪除、PDF/TXT匯出組報告、AI路徑逐欄位fallback（含`mixed`來源標記與`missing[]`提示）皆正確運作。
