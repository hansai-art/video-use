<p align="center">
  <img src="static/video-use-banner.png" alt="video-use" width="100%">
</p>

# video-use

**video-use** 是一套用 Claude Code 剪輯影片的工具，100% 開源。

將原始影片放進資料夾，接著與 Claude Code 對話，即可取得剪輯完成的 `final.mp4`。它適用於真人口播、精華剪輯、教學、旅遊與訪談等各種內容，無須使用預設模板或複雜選單。

可在 [Browser Use Cloud](https://cloud.browser-use.com/v4?utm_campaign=video-use-use-in-cloud&utm_source=github) 試用 video-use。

## 它能做什麼？

- **移除贅詞與口誤**：例如 `umm`、`uh`，以及講到一半重新開始的片段。
- **自動調色**每一段影片：可使用溫暖電影感、自然清晰風格，或任何自訂的 FFmpeg 效果鏈。
- 每個剪接點都加入 **30 毫秒音訊淡入淡出**，避免出現爆音。
- 依你的風格 **直接燒錄字幕**：預設每組兩個字、全大寫，也可完整自訂。
- 透過 [HyperFrames](https://github.com/heygen-com/hyperframes)、[Remotion](https://www.remotion.dev/)、[Manim](https://www.manim.community/) 或 PIL **產生動畫覆蓋層**；每一支動畫由不同子代理平行製作。
- 在交付前，針對每個剪接點 **自動檢查輸出成品**。
- 將工作階段記錄保存至 `project.md`，下週可從上次的進度繼續。

## 安裝提示詞

將以下內容貼到 Claude Code、Codex、Hermes、OpenClaw，或任何能使用終端機的 AI Agent：

```text
請幫我安裝 https://github.com/browser-use/video-use。

請先閱讀 install.md，安裝這個專案、設定 FFmpeg、將技能註冊到我正在使用的 Agent，並設定 ElevenLabs API 金鑰；需要時再請我貼上金鑰。接著閱讀 SKILL.md，了解日常使用方式；也務必閱讀 helpers/，因為剪輯腳本都在這裡。安裝完成後，請不要自行轉錄任何影片；只要告訴我已經準備好，並等待我將影片放入資料夾。
```

Agent 會處理專案下載、依賴套件安裝與技能註冊，並只向你索取一次 ElevenLabs API 金鑰（可在 [elevenlabs.io/app/settings/api-keys](https://elevenlabs.io/app/settings/api-keys) 取得）。

接著，讓 Agent 讀取放有原始拍攝素材的資料夾：

```bash
cd /path/to/your/videos
claude    # 或 codex、hermes 等
```

若要透過自己的 VPS 或 Telegram 隨時剪輯，可經由 [Browser Use Box](https://browser-use.com/bux) 執行 Agent。可觀看[15 秒示範](https://www.tiktok.com/@browser_use/video/7639824093721758989)。

進入工作階段後，輸入：

> 請將這些素材剪成一支產品發布影片。

它會盤點素材、提出剪輯策略、等待你的確認，然後在來源影片旁產生 `edit/final.mp4`。所有輸出檔都會放在 `<videos_dir>/edit/`；技能資料夾會保持乾淨。

## 手動安裝

若想自行安裝，請依序操作：

```bash
# 1. 下載專案，並在 Agent 的技能資料夾建立符號連結
git clone https://github.com/browser-use/video-use ~/Developer/video-use
ln -sfn ~/Developer/video-use ~/.claude/skills/video-use        # Claude Code
# ln -sfn ~/Developer/video-use ~/.codex/skills/video-use       # Codex

# 2. 安裝相依套件
cd ~/Developer/video-use
uv sync                         # 或：pip install -e .
brew install ffmpeg             # 必要
brew install yt-dlp             # 選用，用於下載線上素材

# 3. 加入 ElevenLabs API 金鑰
cp .env.example .env
$EDITOR .env                    # 填入 ELEVENLABS_API_KEY=...
```

## 運作方式

大型語言模型不會從頭到尾觀看影片，而是透過兩層資料來「閱讀」影片，取得精確到單字邊界的剪輯依據。

<p align="center">
  <img src="static/timeline-view.svg" alt="timeline_view 合成檢視：連續畫格、說話者軌道、聲音波形、單字標記與停頓剪輯候選點" width="100%">
</p>

**第 1 層：音訊逐字稿（固定載入）。** 每支來源影片呼叫一次 ElevenLabs Scribe，即可取得單字級時間碼、說話者辨識與音訊事件（`(笑聲)`、`(掌聲)`、`(嘆氣)`）。所有素材會整理為約 12KB 的單一 `takes_packed.md`，這是大型語言模型主要閱讀的內容。

```
## C0103  （片長：43.0 秒，8 個語句片段）
  [002.52-005.36] S0 網頁代理人做的事情，有九成其實都是浪費。
  [006.08-006.74] S0 我們解決了這個問題。
```

**第 2 層：視覺合成圖（依需求產生）。** `timeline_view` 可為任何時間範圍產生一張包含連續畫格、聲音波形與單字標記的 PNG 圖。它只會在需要判斷時使用，例如難以判定的停頓、重錄版本比較或檢查剪接點是否自然。

> 傳統做法：30,000 個畫格 × 1,500 個 Token = **4,500 萬個 Token 的雜訊**。
> Video Use：**12KB 文字 + 少數幾張 PNG 圖**。

這和 browser-use 提供網頁結構化 DOM、而不是只給大型語言模型一張截圖的概念相同，只是這次應用在影片。

## 處理流程

```
轉錄 ──> 整理 ──> 大型語言模型判斷 ──> EDL 剪輯清單 ──> 輸出影片 ──> 自我檢查
                                                                        │
                                                                        └─ 發現問題？修正並重新輸出（最多 3 次）
```

自我檢查會在每個剪接點，對「已輸出的影片」執行 `timeline_view`，用來找出畫面突跳、音訊爆音與字幕被遮住等問題。通過檢查後，你才會看到預覽。

## 設計原則

1. **文字 + 需要時才看的畫面。** 不傾倒大量畫格；逐字稿才是主要工作介面。
2. **聲音優先，畫面跟進。** 剪接候選點來自語句邊界與停頓空檔。
3. **詢問 → 確認 → 執行 → 自我檢查 → 保存紀錄。** 未取得剪輯策略同意，不會直接動剪。
4. **不預設內容類型。** 先看素材、詢問需求，再開始剪輯。
5. **12 條硬性規則，其餘保留創作自由。** 製作正確性不能妥協，美感可依內容決定。

完整的製作規則與剪輯技巧，請參閱 [`SKILL.md`](./SKILL.md)。
