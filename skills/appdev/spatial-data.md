# Oracle Databaseにおける空間データ

## 概要

Oracle Spatial and Graph（旧称 Oracle Spatial）は、Oracle Database内でネイティブな空間データ型、ジオメトリ演算子、および空間索引を提供している。これらはSQLと完全に統合されており、空間クエリは標準的なリレーショナル・データとともに結合、集計、およびクエリ最適化に参加することができる。

Oracle Spatialは `MDSYS` スキーマ上に構築されており、Oracle Spatialオプションが必要である（Enterprise Editionに同梱されており、最新バージョンのStandard Edition 2でもサブセットとして利用可能）。Oracleの空間機能は、**OGC (Open Geospatial Consortium) Simple Features for SQL** 標準に準拠している。

---

## SDO_GEOMETRY型

Oracleのすべての空間データは、`MDSYS` スキーマで定義された `SDO_GEOMETRY` オブジェクト型を使用して格納される。

### SDO_GEOMETRYの構造

```sql
-- SDO_GEOMETRY の定義 (概念図。MDSYS によって定義されている)
CREATE TYPE sdo_geometry AS OBJECT (
    sdo_gtype   NUMBER,      -- ジオメトリ・タイプ・コード
    sdo_srid    NUMBER,      -- 座標参照系 (EPSG コード)
    sdo_point   SDO_POINT_TYPE,   -- 2D/3D 点型用のショートカット
    sdo_elem_info SDO_ELEM_INFO_ARRAY,  -- 要素情報配列
    sdo_ordinates SDO_ORDINATE_ARRAY    -- パックされた座標値配列
);
```

### SDO_GTYPE: ジオメトリ・タイプ・コード

`SDO_GTYPE` は4桁のコード **DLTT** である。
- **D**: 次元数 (2, 3, 4)
- **L**: LRS (線形参照系) メジャー次元 (なしの場合は 0)
- **TT**: ジオメトリ・タイプ (01–07)

| GTYPE | 説明 |
|---|---|
| 2001 | 2D 点 (Point) |
| 3001 | 3D 点 (Point) |
| 2002 | 2D 線分 (Line String) |
| 2003 | 2D ポリゴン (Polygon) |
| 3003 | 3D ポリゴン (Polygon) |
| 2004 | 2D ジオメトリ・コレクション |
| 2005 | 2D 複数点 (MultiPoint) |
| 2006 | 2D 複数線分 (MultiLine) |
| 2007 | 2D 複数ポリゴン (MultiPolygon) |

---

## 一般的なジオメトリ・サブタイプ

### 点 (Point)

```sql
-- SDO_POINT を使用した 2D 点 (点データに対して最も高速かつシンプル)
-- 形式: SDO_GEOMETRY(gtype, srid, SDO_POINT_TYPE(x, y, z_or_null), null, null)
SELECT SDO_GEOMETRY(
    2001,                           -- 2D 点
    4326,                           -- WGS84 座標系 (GPS)
    SDO_POINT_TYPE(-122.4194, 37.7749, NULL),  -- サンフランシスコ (経度, 緯度)
    NULL,
    NULL
) AS sf_location
FROM DUAL;

-- 3D 点
SELECT SDO_GEOMETRY(
    3001,            -- 3D 点
    4326,
    SDO_POINT_TYPE(-122.4194, 37.7749, 52.0),  -- 標高 (メートル) 付き
    NULL,
    NULL
) FROM DUAL;
```

### 線分 (Line String)

```sql
-- 2D 線分 (ルートや道路セグメント)
-- SDO_ELEM_INFO: (開始オフセット, etype, interpretation)
--   etype 2 = 線分, interpretation 1 = 直線セグメント
SELECT SDO_GEOMETRY(
    2002,           -- 2D 線分
    4326,           -- WGS84
    NULL,
    SDO_ELEM_INFO_ARRAY(1, 2, 1),                -- 1つの線分、直線セグメント
    SDO_ORDINATE_ARRAY(
        -122.4194, 37.7749,   -- 点 1 (開始)
        -122.4094, 37.849,    -- 点 2
        -122.3994, 37.7749    -- 点 3 (終了)
    )
) AS route
FROM DUAL;
```

### ポリゴン (Polygon)

```sql
-- 単純な 2D ポリゴン (閉じたリング、終点 = 始点)
-- etype 1003 = 外部ポリゴン・リング, interpretation 1 = 直線セグメント
SELECT SDO_GEOMETRY(
    2003,           -- 2D ポリゴン
    4326,
    NULL,
    SDO_ELEM_INFO_ARRAY(1, 1003, 1),    -- 外部リング、直線セグメント
    SDO_ORDINATE_ARRAY(
        -122.45, 37.75,   -- 南西角
        -122.40, 37.75,   -- 南東角
        -122.40, 37.80,   -- 北東角
        -122.45, 37.80,   -- 北西角
        -122.45, 37.75    -- リングを閉じる (始点と同じ)
    )
) AS sf_district
FROM DUAL;

-- ホール（穴）を持つポリゴン (ドーナツ型)
SELECT SDO_GEOMETRY(
    2003,
    4326,
    NULL,
    SDO_ELEM_INFO_ARRAY(
        1, 1003, 1,   -- 外部リングが座標位置 1 から開始
        11, 2003, 1   -- 内部リング (ホール) が座標位置 11 から開始
    ),
    SDO_ORDINATE_ARRAY(
        -- 外部リング (5点 = 10座標)
        0, 0,  10, 0,  10, 10,  0, 10,  0, 0,
        -- 内部リング / ホール (5点 = 10座標)
        2, 2,   8, 2,   8,  8,  2,  8,  2, 2
    )
) AS donut_polygon
FROM DUAL;
```

---

## 空間表のセットアップ

### 表の作成と USER_SDO_GEOM_METADATA への登録

**空間索引を作成する前に、すべての空間表のエントリを `USER_SDO_GEOM_METADATA` に追加する必要がある。** このメタデータによって、有効な座標範囲が定義される。

```sql
-- 表の作成
CREATE TABLE store_locations (
    store_id    NUMBER PRIMARY KEY,
    store_name  VARCHAR2(100),
    city        VARCHAR2(50),
    location    MDSYS.SDO_GEOMETRY
);

-- 空間メタデータの登録 (空間索引作成前に必須)
-- diminfo: 次元情報の配列: (名前, 最小値, 最大値, 許容差)
-- tolerance: 座標単位における最小の有意な距離 (WGS84 の場合は度)
INSERT INTO user_sdo_geom_metadata (table_name, column_name, diminfo, srid)
VALUES (
    'STORE_LOCATIONS',
    'LOCATION',
    SDO_DIM_ARRAY(
        SDO_DIM_ELEMENT('LONGITUDE', -180, 180, 0.00001),  -- 度単位で約 1 メートル
        SDO_DIM_ELEMENT('LATITUDE',   -90,  90, 0.00001)
    ),
    4326  -- WGS84 (GPS 座標)
);
COMMIT;

-- サンプルデータの挿入
INSERT INTO store_locations VALUES (1, 'SF Downtown', 'San Francisco',
    SDO_GEOMETRY(2001, 4326, SDO_POINT_TYPE(-122.4194, 37.7749, NULL), NULL, NULL));

INSERT INTO store_locations VALUES (2, 'Oakland Uptown', 'Oakland',
    SDO_GEOMETRY(2001, 4326, SDO_POINT_TYPE(-122.2711, 37.8044, NULL), NULL, NULL));

INSERT INTO store_locations VALUES (3, 'San Jose Center', 'San Jose',
    SDO_GEOMETRY(2001, 4326, SDO_POINT_TYPE(-121.8863, 37.3382, NULL), NULL, NULL));

COMMIT;
```

---

## 空間索引

Oracleの空間索引は Rツリー索引（特定の場合はクワッドツリー）である。`INDEXTYPE IS MDSYS.SPATIAL_INDEX` 構文を使用して作成する。

```sql
-- 空間索引の作成 (事前にメタデータの登録が必要)
CREATE INDEX idx_store_locations_geom
    ON store_locations (location)
    INDEXTYPE IS MDSYS.SPATIAL_INDEX
    PARAMETERS ('sdo_indx_dims=2');

-- 3D 空間データ用
CREATE INDEX idx_buildings_3d
    ON buildings (geom_col)
    INDEXTYPE IS MDSYS.SPATIAL_INDEX_V2
    PARAMETERS ('sdo_indx_dims=3');

-- 索引作成の確認
SELECT index_name, status, ityp_owner, ityp_name
FROM   user_indexes
WHERE  table_name = 'STORE_LOCATIONS';
```

### 空間索引の再構築 (REBUILD)

```sql
-- 空間索引の再構築 (大量ロード後など)
ALTER INDEX idx_store_locations_geom REBUILD;

-- 空間索引の妥当性確認
SELECT sdo_index_name, sdo_index_type, sdo_index_status
FROM   mdsys.sdo_index_info_table
WHERE  sdo_index_table_name = 'STORE_LOCATIONS';
```

---

## 空間演算子

Oracle Spatialは、主要な空間述語に**演算子**（関数ではない）を使用する。オプティマイザはこれらの演算子を使用して空間索引を活用する。

### SDO_RELATE: 一般的な位相関係

`SDO_RELATE` は、9交点モデル (DE-9IM) を使用して2つのジオメトリ間の位相関係をテストする。

```sql
-- 地区境界ポリゴン内にあるすべての店舗を検索
SELECT s.store_id, s.store_name
FROM   store_locations s,
       district_boundaries d
WHERE  d.district_name = 'Bay Area'
  AND  SDO_RELATE(
            s.location,          -- ジオメトリ 1 (索引付きの列)
            d.boundary,          -- ジオメトリ 2
            'mask=INSIDE'        -- 関係マスク
       ) = 'TRUE';
```

**関係マスク:**

| マスク | 説明 |
|---|---|
| `TOUCH` | 境界が接しているが、内部は交差していない |
| `OVERLAPBDYDISJOINT` | 境界は互いに離れているが、重なっている |
| `OVERLAPBDYINTERSECT` | 境界が交差しており、重なっている |
| `EQUAL` | 幾何学的に等しい |
| `INSIDE` | ジオメトリ 1 がジオメトリ 2 の内部にある |
| `COVEREDBY` | ジオメトリ 1 がジオメトリ 2 に覆われている (または内部にある) |
| `CONTAINS` | ジオメトリ 1 がジオメトリ 2 を含んでいる |
| `COVERS` | ジオメトリ 1 がジオメトリ 2 を覆っている (または含んでいる) |
| `ANYINTERACT` | 何らかの相互作用がある (最も一般的に使用される) |
| `ON` | ジオメトリ 1 がジオメトリ 2 の境界上にある |

```sql
-- ANYINTERACT: 互いに接している、重なっている、または含んでいるジオメトリを検索
SELECT s.store_id, s.store_name
FROM   store_locations s,
       flood_zones f
WHERE  f.risk_level = 'HIGH'
  AND  SDO_RELATE(s.location, f.boundary, 'mask=ANYINTERACT') = 'TRUE';

-- 複数のマスクを + で組み合わせる
SELECT * FROM parcel_map p, utility_lines u
WHERE  SDO_RELATE(p.geom, u.geom, 'mask=TOUCH+OVERLAPBDYINTERSECT') = 'TRUE';
```

### SDO_WITHIN_DISTANCE: 近接検索

```sql
-- 指定された点 (例: 顧客の場所) から 5 km 以内の店舗を検索
SELECT s.store_id, s.store_name, s.city
FROM   store_locations s
WHERE  SDO_WITHIN_DISTANCE(
            s.location,                                              -- 索引付きジオメトリ
            SDO_GEOMETRY(2001, 4326,
                SDO_POINT_TYPE(-122.4000, 37.7700, NULL), NULL, NULL),  -- クエリ地点
            'distance=5 unit=km'                                     -- 距離の指定
       ) = 'TRUE';

-- 実際の実距離で結果をソート
SELECT s.store_id, s.store_name,
       SDO_GEOM.SDO_DISTANCE(
            s.location,
            SDO_GEOMETRY(2001, 4326, SDO_POINT_TYPE(-122.4000, 37.7700, NULL), NULL, NULL),
            0.001,   -- 許容差
            'unit=km'
       ) AS distance_km
FROM   store_locations s
WHERE  SDO_WITHIN_DISTANCE(
            s.location,
            SDO_GEOMETRY(2001, 4326, SDO_POINT_TYPE(-122.4000, 37.7700, NULL), NULL, NULL),
            'distance=5 unit=km'
       ) = 'TRUE'
ORDER  BY distance_km;
```

### SDO_NN: 最近傍検索 (Nearest Neighbor)

```sql
-- 顧客の場所に最も近い3つの店舗を検索
SELECT s.store_id, s.store_name,
       SDO_NN_DISTANCE(1) AS distance_meters
FROM   store_locations s
WHERE  SDO_NN(
            s.location,
            SDO_GEOMETRY(2001, 4326, SDO_POINT_TYPE(-122.4000, 37.7700, NULL), NULL, NULL),
            'sdo_num_res=3 unit=meter',
            1          -- 相関番号 (SDO_NN_DISTANCE の引数と一致させる必要がある)
       ) = 'TRUE'
ORDER  BY distance_meters;

-- 追加のフィルタを伴う SDO_NN (開店している店舗のみ)
SELECT s.store_id, s.store_name, SDO_NN_DISTANCE(1) AS dist
FROM   store_locations s
WHERE  SDO_NN(s.location,
              SDO_GEOMETRY(2001, 4326, SDO_POINT_TYPE(-122.4, 37.77, NULL), NULL, NULL),
              'sdo_num_res=10', 1) = 'TRUE'
  AND  s.is_open = 'Y'
ORDER  BY dist
FETCH FIRST 3 ROWS ONLY;
```

### SDO_CONTAINS および SDO_INSIDE

```sql
-- ポリゴン内のすべての点を検索
SELECT s.store_id, s.store_name
FROM   store_locations s,
       sales_territories t
WHERE  t.territory_id = 7
  AND  SDO_CONTAINS(t.boundary, s.location) = 'TRUE';

-- SDO_INSIDE: CONTAINS の逆
SELECT t.territory_name
FROM   store_locations s,
       sales_territories t
WHERE  s.store_id = 42
  AND  SDO_INSIDE(s.location, t.boundary) = 'TRUE';
```

---

## SDO_GEOM 関数: 計測と演算

```sql
-- 2点間の距離を計算
SELECT SDO_GEOM.SDO_DISTANCE(
    SDO_GEOMETRY(2001, 4326, SDO_POINT_TYPE(-122.4194, 37.7749, NULL), NULL, NULL),
    SDO_GEOMETRY(2001, 4326, SDO_POINT_TYPE(-118.2437, 34.0522, NULL), NULL, NULL),
    0.001,         -- 許容差
    'unit=km'
) AS sf_to_la_km
FROM DUAL;

-- ポリゴンの面積を計算
SELECT SDO_GEOM.SDO_AREA(
    SDO_GEOMETRY(
        2003, 4326, NULL,
        SDO_ELEM_INFO_ARRAY(1, 1003, 1),
        SDO_ORDINATE_ARRAY(-122.45, 37.75, -122.40, 37.75,
                           -122.40, 37.80, -122.45, 37.80, -122.45, 37.75)
    ),
    0.001,         -- 許容差
    'unit=sq_km'   -- 平方キロメートル
) AS area_sq_km
FROM DUAL;

-- 長さ/周囲長を計算
SELECT SDO_GEOM.SDO_LENGTH(geom, 0.001, 'unit=km') AS length_km
FROM   road_segments
WHERE  road_id = 101;

-- バッファ: ジオメトリから一定距離にあるポリゴンを作成
SELECT SDO_GEOM.SDO_BUFFER(
    location,
    5000,         -- 5000 メートル
    0.001         -- 許容差
) AS five_km_buffer
FROM   store_locations
WHERE  store_id = 1;

-- ジオメトリの結合 (Union)
SELECT SDO_GEOM.SDO_UNION(geom_a, geom_b, 0.001) AS merged_geom
FROM   (SELECT a.boundary AS geom_a, b.boundary AS geom_b
        FROM   sales_territories a, sales_territories b
        WHERE  a.territory_id = 1 AND b.territory_id = 2);

-- 交差 (Intersection)
SELECT SDO_GEOM.SDO_INTERSECTION(
    polygon_a, polygon_b, 0.001
) AS intersection_geom
FROM   geometry_pairs;
```

---

## 座標参照系 (SRID)

**SRID (空間参照識別子: Spatial Reference Identifier)** は座標系を定義する。Oracleは定義を `MDSYS.SDO_COORD_REF_SYSTEM` に格納している。

```sql
-- 一般的な SRID
-- 4326  = WGS84 (GPS, 度単位の経度/緯度) — 最も一般的
-- 3857  = Web メルカトル (Google Maps, OpenStreetMap) — 投影座標系, メートル単位
-- 27700 = British National Grid (メートル単位, 英国)
-- 32610 = WGS84 / UTM Zone 10N (メートル単位, 米国西部)

-- 座標系の検索
SELECT srid, coord_ref_sys_name, coord_ref_sys_kind
FROM   mdsys.sdo_coord_ref_system
WHERE  srid IN (4326, 3857, 32610);

-- 座標系間の変換
SELECT SDO_CS.TRANSFORM(
    location,
    3857    -- 4326 (WGS84) から 3857 (Web メルカトル) に変換
) AS location_web_mercator
FROM   store_locations
WHERE  store_id = 1;

-- 格納されているデータの座標系を確認
SELECT s.store_id, s.location.sdo_srid
FROM   store_locations s;
```

---

## GeoJSON との統合

```sql
-- SDO_GEOMETRY を GeoJSON に変換
SELECT SDO_UTIL.TO_GEOJSON(location) AS geojson
FROM   store_locations
WHERE  store_id = 1;
-- 戻り値: {"type":"Point","coordinates":[-122.4194,37.7749]}

-- GeoJSON を SDO_GEOMETRY に変換
SELECT SDO_UTIL.FROM_GEOJSON(
    '{"type":"Point","coordinates":[-122.4194,37.7749]}'
) AS location
FROM DUAL;

-- REST API 用の完全な Feature Collection 生成
SELECT JSON_ARRAYAGG(
    JSON_OBJECT(
        'type' VALUE 'Feature',
        'id'   VALUE store_id,
        'geometry' VALUE JSON(SDO_UTIL.TO_GEOJSON(location)),
        'properties' VALUE JSON_OBJECT(
            'name' VALUE store_name,
            'city' VALUE city
        )
    )
) AS geojson_collection
FROM   store_locations;
```

---

## ベスト・プラクティス

- **空間索引を作成する前に、必ず `USER_SDO_GEOM_METADATA` を登録する。** メタデータは、有効な座標範囲と許容差を定義する。
- 汎用的な地理データ（GPS 座標）には **WGS84 (SRID=4326)** を使用する。正確なメートル単位の距離が必要な場合は、投影座標系 (UTM, 平面直角座標系など) を使用する。
- **許容差を適切に設定する**: 地理データの経緯度では約 0.00001 度 (≈1 メートル)、メートル単位の投影データでは 0.001 に設定する。許容差が小さすぎると等価判定に失敗し、大きすぎると近接する機能が混同される。
- 空間索引を活用するために、WHERE 句では空間関数 (`SDO_GEOM.*`) ではなく、**空間演算子 (`SDO_RELATE`, `SDO_NN`)** を使用する。
- 頻繁に比較されるジオメトリ・ペアについては、あらかじめ距離を計算しておき、標準的な NUMBER 列にBツリー索引を付けて格納することを検討する。
- 最近傍検索には、広い半径の `SDO_WITHIN_DISTANCE` よりも、索引スキャンの効率が良い **`SDO_NN`** を使用する。
- 巨大な空間表は、地域（州や国など）ごとに**パーティション化**し、空間クエリでのパーティション・プルーニングを可能にする。
- 挿入前に `SDO_GEOM.VALIDATE_GEOMETRY_WITH_CONTEXT` を使用して**ジオメトリを検証**する。

```sql
-- 挿入前のジオメトリ検証
DECLARE
    v_result VARCHAR2(100);
BEGIN
    v_result := SDO_GEOM.VALIDATE_GEOMETRY_WITH_CONTEXT(
        SDO_GEOMETRY(2003, 4326, NULL,
            SDO_ELEM_INFO_ARRAY(1, 1003, 1),
            SDO_ORDINATE_ARRAY(0,0, 1,0, 1,1, 0,1, 0,0)
        ),
        0.001
    );
    IF v_result != 'TRUE' THEN
        RAISE_APPLICATION_ERROR(-20010, 'Invalid geometry: ' || v_result);
    END IF;
END;
```

---

## よくある間違い

### 間違い 1: メタデータなしで空間索引を作成する

```sql
-- 誤り: ORA-13203 で失敗する
CREATE INDEX idx_spatial ON stores(location) INDEXTYPE IS MDSYS.SPATIAL_INDEX;

-- 正解: メタデータを先に挿入し、その後で索引を作成する
INSERT INTO user_sdo_geom_metadata VALUES (...);
COMMIT;
CREATE INDEX idx_spatial ON stores(location) INDEXTYPE IS MDSYS.SPATIAL_INDEX;
```

### 間違い 2: 緯度と経度を入れ違える

Oracle の WGS84 (SRID=4326) 用 SDO_GEOMETRY は、上の例にある通り **(経度, 緯度)** の順序を使用する（緯度, 経度ではない）。これは数学的な (x, y) 座標の慣習および OGC/GeoJSON 標準と一致しているが、口頭で座標を説明する際とは逆になることが多いため注意が必要。

```sql
-- 誤り: 緯度を先に指定
SDO_POINT_TYPE(37.7749, -122.4194, NULL)  -- これは大西洋上を指してしまう

-- 正解: 経度を先、次に緯度
SDO_POINT_TYPE(-122.4194, 37.7749, NULL)  -- サンフランシスコ
```

### 間違い 3: WHERE 句で SDO_GEOM 関数を使用する (索引が効かない)

```sql
-- 誤り: SDO_GEOM.SDO_DISTANCE は空間索引を使用しない
WHERE SDO_GEOM.SDO_DISTANCE(s.location, :point, 0.001) < 5000;

-- 正解: 索引によって加速される SDO_WITHIN_DISTANCE を使用する
WHERE SDO_WITHIN_DISTANCE(s.location, :point, 'distance=5000') = 'TRUE'
```

### 間違い 4: ポリゴン・リングを閉じていない

ポリゴンの最初と最後の座標ペアは、リングを閉じるために同一である必要がある。閉じられていないリングは無効なジオメトリとなる。

### 間違い 5: 座標系に対して不適切な許容差

メートル単位の投影座標系（値が大きい）に対して、非常に小さな許容差（例：0.000001）を使用すると、ほとんどの演算で予期しない結果が返される。許容差は SRID の単位スケールに合わせて設定すること。

---

## Oracleバージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または23aiとしてマークされた機能は、Oracle Database 26ai対応機能として扱う。混在バージョン構成の場合は、19c互換の代替案を保持すること。
- 両方のバージョンをサポートする環境では、リリースの更新によってデフォルトや非推奨が異なる可能性があるため、19cと26aiの両方で構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 19c Spatial and Graph 開発者ガイド (SPATL)](https://docs.oracle.com/en/database/oracle/oracle-database/19/spatl/)
- [Oracle Database 19c SQL Multimedia and Image リファレンス](https://docs.oracle.com/en/database/oracle/oracle-database/19/imref/)
