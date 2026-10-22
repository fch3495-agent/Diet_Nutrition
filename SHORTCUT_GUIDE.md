# iOS捷徑搭建教學 — 拍照記錄飲食營養

拍照 → Claude API辨識食物 → 查衛福部資料庫算精確營養值(查無則用AI估算) → 寫入Apple Health。

## 前置準備

1. 到 [console.anthropic.com](https://console.anthropic.com) 申請API key,信用卡綁定、儲值(US$5~10夠用很久)
   - 一天3張照片、一年約US$2~3(用Haiku模型估算),成本可忽略
2. 確認iPhone上「捷徑」App、「健康」App都已設定好
3. 本文用到的資料庫Raw網址:
   ```
   https://raw.githubusercontent.com/fch3495-agent/Diet_Nutrition/main/tfnd_simplified.json
   ```

## 捷徑動作順序

### 步驟1-2:拍照 + 轉Base64

- 動作「拍攝照片」或「選取照片」
- 動作「Base64編碼」,輸入接上一步照片

### 實測踩雷記錄(2026-10-22)

直接在「取得URL內容」動作的要求本文裡用JSON欄位建構器疊巢狀辭典/列表,**實測會讓捷徑App當機/卡死**(疊兩三層辭典後插變數必當)。改用下面這個方法穩定可行:

1. 用獨立的「**文字**」動作,把整段JSON**當純文字手打**出來(不用巢狀欄位UI)
2. 打到Base64資料要插入的地方,用「選取變數」插入Base64編碼動作的輸出
3. 「取得URL內容」動作的**要求本文選「檔案」**,欄位插入這個「文字」動作的完整輸出(不要選JSON選項)

另外踩到的雷:**魔術變數常抓不到剛做的Base64編碼結果**(選取變數清單只顯示不相關的URL建議、或清單卡在某個舊變數)。這是捷徑App索引卡住的問題,解法:**把「以Base64編碼」那個動作整個刪掉、重新搜尋新增一次**,通常這樣就能讓新建的動作被系統重新正常索引,之後選取變數就抓得到了。純重開App不一定有用,要刪掉重建那個動作本身。

### 步驟3-4:呼叫Claude API辨識食物

**辨識用Prompt(放進Request Body的text內容):**

```
你是食物辨識與營養分析助手。請分析這張照片中的食物,只回傳以下JSON格式,不要加任何其他文字或說明:

{
  "food_name": "食物中文名稱(越精確越好,例如:白米飯、紅燒肉、青菜炒蛋)",
  "estimated_weight_g": 份量估計整數(單位克),
  "calories_kcal": 估計熱量整數,
  "protein_g": 估計蛋白質(到小數點一位),
  "carbs_g": 估計碳水化合物(到小數點一位),
  "fat_g": 估計脂肪(到小數點一位),
  "confidence": "high/medium/low"
}

若照片中有多種食物,food_name用「、」分隔列出,並將所有營養值加總為整份餐點的總量。
```

**動作「取得URL內容」設定:**

- URL:`https://api.anthropic.com/v1/messages`
- 方法:POST
- Headers:
  - `x-api-key`: 你的API key
  - `anthropic-version`: `2023-06-01`
  - `content-type`: `application/json`
- Request Body:**選「檔案」**,內容插入下面這個獨立「文字」動作的輸出(不要用JSON欄位建構器,見上方踩雷記錄)

**獨立「文字」動作內容**(在這個「取得URL內容」動作**之前**新增,純文字手打,插入變數的地方用「選取變數」):

```json
{"model":"claude-haiku-5-5","max_tokens":300,"messages":[{"role":"user","content":[{"type":"image","source":{"type":"base64","media_type":"image/jpeg","data":"<這裡插入Base64編碼動作的輸出變數>"}},{"type":"text","text":"<上面那段prompt,整段貼入,prompt本身不含雙引號可直接貼>"}]}]}
```

### 步驟5:解析回傳JSON

- 動作「取得字典值」,從API回傳結果取 `content[0].text`,再「從輸入取得字典」解析成JSON
- 從中取出 `food_name`、`estimated_weight_g`、`calories_kcal`、`protein_g`、`carbs_g`、`fat_g` 存成變數(當AI估算備援值)

### 步驟6:查衛福部資料庫比對

- 動作「取得URL內容」GET抓資料庫Raw網址(見前置準備),回傳是一個陣列,每筆含 `name`/`alias`/`calories_kcal`/`protein_g`/`fat_g`/`carbs_g`
- 用「篩選[捷徑的清單]」或「重複處理清單中的每一項目」,比對 `name` 或 `alias` 是否包含 `food_name`(字串包含,關鍵字比對)
- 比對到→取出該筆 `calories_kcal` 等四項(這是「每100克」數值)

### 步驟7:算實際攝取量

- 若步驟6比對到資料庫數值:
  ```
  實際熱量 = 資料庫每100克熱量 × (estimated_weight_g / 100)
  ```
  蛋白/脂肪/碳水同理換算
- 若步驟6比對不到:直接用步驟5的AI估算值(已經是整份的估計,不用再乘份量)

### 步驟8:寫入Apple Health

依序用以下Shortcuts內建Health動作寫入步驟7算出的最終數值:

- 「記錄飲食能量」(Log Dietary Energy) — 單位kcal
- 「記錄蛋白質」(Log Protein) — 單位g
- 「記錄碳水化合物」(Log Carbohydrates) — 單位g
- 「記錄脂肪」(Log Total Fat) — 單位g

每個動作記得選對應的變數值,日期用「現在」。

## 流程圖

```
拍照 → Base64 → Claude API辨識(食物名稱+估重+AI估算營養)
                        │
                        ▼
              查衛福部資料庫(用食物名稱比對)
                   │              │
               比對成功        比對失敗
                   │              │
         資料庫數值×份量比例   直接用AI估算值
                   │              │
                   └──────┬───────┘
                          ▼
                  寫入Apple Health
          (熱量/蛋白/碳水/脂肪四項)
```

## 除錯提醒

- API回傳400/401:檢查API key是否正確、`anthropic-version` header有沒有帶
- 資料庫抓不到/比對永遠失敗:檢查Raw網址能不能在瀏覽器直接打開看到JSON(repo已是public,應該能打開)
- 中文食物名稱比對不到:資料庫用的是正式食品名稱(如「白米(蓬萊米)」),口語說法(如「白飯」)可能比對不到,可以放寬成「取前2個字比對」或接受失敗退回AI估算
