# Diet_Nutrition

拍照辨識食物營養成分用的資料庫,配合iOS捷徑(Shortcuts) + Claude API辨識食物,查表拿精確營養數值,寫入Apple Health。

## 資料來源

`tfnd_simplified.json` 是衛福部食藥署「食品營養成分資料庫」的精簡版。

- 原始資料:[data.gov.tw 食品營養成分資料集(編號8543)](https://data.gov.tw/dataset/8543)
- 原始下載(JSON,約130MB,長表格式一食物多列):https://data.fda.gov.tw/data/opendata/export/20/json
- 授權:政府資料開放授權條款第1版

## 精簡邏輯

原始資料是「一食物多列」(每個分析項一列),本檔轉成「一食物一列」,只保留四項核心營養素(每100克含量):

| 欄位 | 說明 |
|---|---|
| `name` | 樣品名稱 |
| `alias` | 俗名(比對用) |
| `category` | 食品分類 |
| `calories_kcal` | 熱量(每100克) |
| `protein_g` | 粗蛋白(每100克) |
| `fat_g` | 粗脂肪(每100克) |
| `carbs_g` | 總碳水化合物(每100克) |
| `code` | 整合編號 |

共2213筆(僅收錄有熱量數據的食物),379KB。

## 用途

iOS捷徑拍照後,呼叫Claude API辨識食物名稱+估計份量,再抓這個repo的Raw JSON網址,用食物名稱關鍵字比對本資料庫:

```
https://raw.githubusercontent.com/fch3495-agent/Diet_Nutrition/main/tfnd_simplified.json
```

比對有結果→用資料庫數值算出實際攝取量;比對無結果→退回用Claude的AI估算當備援。

## 更新資料庫

衛福部資料庫不定期更新,若要重新整理:

1. 重新下載 https://data.fda.gov.tw/data/opendata/export/20/json (ZIP壓縮)
2. 解壓後,依「分析項分類=一般成分」篩出 熱量/粗蛋白/粗脂肪/總碳水化合物 四項,依整合編號(`code`)聚合成一食物一列
3. 輸出覆蓋 `tfnd_simplified.json`
