# happy-path-osm

台灣的靜態圖資檔案，透過 GitHub Pages 以 HTTP Range 提供。
**沒有後端、沒有程式在跑**——這個 repo 只是一組不可變的檔案。

檔案多大，與一次查詢抓多少無關——FlatGeobuf 的檔頭裡有 packed R-tree，客戶端先讀索引再只抓需要的位元組。
查台北車站 5 km 的 bbox：11.2 MB 的 `roads.fgb` 只發 5 次 Range 請求、抓 1.3 MB。

## 內容

```
osm/2026-09/
├── manifest.json        契約：版本、coverage、每個檔的大小/sha256/envelope/來源
├── roads.fgb            43,259  motorway/trunk/primary/secondary（含 _link）
├── roads_major.fgb      11,257  只有 motorway/trunk，全台視野的粗層
├── waterways.fgb         8,330  river / canal 中心線
├── water.fgb            58,322  natural=water 面
├── coastlines.fgb        1,787  海岸線（裁過的縫不成閉環，一律整層抓）
├── peaks.fgb             5,541  natural=peak 點，帶海拔
├── buildings.fgb       590,891  建物質心點 + 底面積 / 樓層 / 類型 / 名稱
└── tw.dem.bin.gz               3″（≈93 m）int16 高程網格 3360×4320，gzip 6.2 MB（原 27.7 MB）
```

合計 120 MB，全部 `EPSG:4326`。目錄名帶日期是刻意的：**不覆蓋舊版，要換就開新目錄**。

`manifest.json` 的 `coverage` = `119.3, 21.9034, 122.0035, 25.4`，是 `roads.fgb` 的 envelope
與 DEM bbox 的交集（金門馬祖有路但在 DEM 之外）。消費端應該讀這個值，不要各自寫死範圍。

## 資料來源

### 七個 `.fgb` ← OpenStreetMap

原始檔向 [Geofabrik](https://download.geofabrik.de/asia/taiwan.html) 取得——它每天從 OSM 主庫
切出各地區的 `.osm.pbf`，是拿整區 OSM 資料的標準做法（OSM 的 API 只給小範圍編輯用，不要拿來抓全區）。

```bash
curl -L -o taiwan-260914.osm.pbf https://download.geofabrik.de/asia/taiwan-latest.osm.pbf   # 326 MB
```

這裡用的是 2026-09-14 下載的版本。`.pbf` 再經篩選 → GeoJSONSeq → 建空間索引 → `.fgb`，見下方「重新產生」。

### `tw.dem.bin.gz` ← Copernicus DEM GLO-30

[AWS Open Data](https://registry.opendata.aws/copernicus-dem/) 的公開 bucket
`copernicus-dem-30m.s3.amazonaws.com`，取 N21–N25 × E119–E122 共 14 片 1°×1° COG，
以 3×3 中位數重取樣成 3 角秒網格。腳本會自動下載，不必手動抓。

## 怎麼用

```js
import { HttpReader } from 'flatgeobuf/lib/mjs/http-reader.js';

const BASE = 'https://freddycro.github.io/happy-path-osm/osm/2026-09/';
const reader = await HttpReader.open(`${BASE}roads.fgb`, false);
for await (const { feature } of reader.selectBbox({ minX: 121.47, minY: 25.0, maxX: 121.57, maxY: 25.09 })) {
  // 只有這個 bbox 內的路被下載
}
```

DEM 是整份下載（6.2 MB，之後永遠快取）：

```js
let buf = await (await fetch(`${BASE}tw.dem.bin.gz`)).arrayBuffer();
// ⚠️ 有些 server（GitHub Pages 的 Fastly、Vite 的 sirv）看到 .gz 會自己加 Content-Encoding: gzip，
// 瀏覽器已透明解掉——再解一次就壞。看 magic bytes 決定：
const h = new Uint8Array(buf, 0, 2);
if (h[0] === 0x1f && h[1] === 0x8b) {
  buf = await new Response(new Blob([buf]).stream().pipeThrough(new DecompressionStream('gzip'))).arrayBuffer();
}
```

驗證 Range 有生效（**用 GET，不要用 `curl -I`**——HEAD 會忽略 Range 回 200 而誤判）：

```bash
curl -s -o /dev/null -D - -H "Range: bytes=0-1023" \
  "https://freddycro.github.io/happy-path-osm/osm/2026-09/roads.fgb" | grep -i "^HTTP/\|content-range\|access-control"
# 要看到 206 Partial Content、Content-Range、Access-Control-Allow-Origin: *
```

## 授權

| 檔案 | 授權 | 必須標示 |
|---|---|---|
| 七個 `.fgb` | [ODbL 1.0](https://www.openstreetmap.org/copyright)（衍生資料庫，含 share-alike） | `© OpenStreetMap contributors` |
| `tw.dem.bin.gz` | Copernicus 條款（須標示，**不是** ODbL、無 share-alike） | `Copernicus DEM GLO-30 © DLR e.V. 2010-2014, © Airbus Defence and Space GmbH 2014-2018, provided under COPERNICUS by the European Union and ESA` |
| 文件 | CC0 1.0 | — |

用這些資料的頁面必須顯示上述兩段字串；它們也寫在 `manifest.json` 的 `sources` 與 DEM 檔頭的
`attribution` 裡，消費端該從那裡讀而不是自己拼。

拿資料去渲染圖像或 3D 場景屬於 Produced Work，只需標示出處；只有散布資料本身的修改版才需要一併以
ODbL 釋出。完整說明見 [LICENSE](LICENSE)。

## 重新產生

腳本不在這個 repo，在 [happy-path](https://github.com/FreddyCro/happy-path) 的
`soft-island/scripts/` 與 `geodata/scripts/`：

```bash
# 1. pbf 人工下載一次（見上方「資料來源」），放 temp/
# 2. DEM（自動抓 Copernicus 14 片）
cd soft-island && node scripts/dem-to-grid.mjs
# 3. pbf → 中間格式 → GeoJSONSeq → 帶索引的 FGB → 壓 DEM → manifest（含斷言）
node scripts/pbf-to-intermediate.mjs --in ../temp/taiwan-260914.osm.pbf --out ../temp/tw-intermediate
cd ../geodata
node scripts/to-geojsonseq.mjs --ndjson-dir ../temp/tw-intermediate --out build/geojsonl-tw
node scripts/to-fgb.mjs --in build/geojsonl-tw --out public/data
node scripts/pack-dem.mjs
node scripts/manifest.mjs --version 2026-09
# 4. 複製到這個 repo 的 osm/<版本>/ 並 commit
```

全程約 5 分鐘，大部分時間在下載。

兩個踩過的坑：

- **索引一定要用 GDAL 寫。** `flatgeobuf` 的 JS `serialize()` 把 `indexNodeSize` 寫死 0，不產生空間索引，
  那種 `.fgb` 只能整份讀。`to-fgb.mjs` 改用 gdal3.js（GDAL 的 WebAssembly 版，零安裝）跑
  `ogr2ogr -f FlatGeobuf -lco SPATIAL_INDEX=YES`；`manifest.mjs` 會再斷言一次 `indexNodeSize ≠ 0`。
- **不用 Git LFS**，GitHub Pages 不支援、會把 pointer 檔原樣送出。二進位直接進 git，
  最大的 `buildings.fgb` 77 MB 在 GitHub 的 100 MB 上限內。
