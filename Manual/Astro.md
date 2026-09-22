# Astro.md

バンドルは以下の構造となっている

## 用語

- バンドル: 複数のアセットと設定情報をまとめたデータ
- アセット: door、floor など、Astro が書き出した単位
- アセット名: Prefab や json を読み込むときに指定する文字列
- カラバリ: アセット1個につき、上限なく作れる見た目違い
  - 形状や役割が一緒だkがメッシュやマテリアルの構造が違うもの
  - 00〜99までの最大100個存在する。

## Astroのバンドル構造

Addressableで出力したバンドルデータの一覧

- バンドル 
  - カタログ(アセット名: astro) AstroCatalog型
  - アセット名ごと(アセット名: door、floor等)
    - prefab(最大 99 まで)　GameObject型
      - 00.prefab
      - 01.prefab
      〜
      - 99.prefab
    - json(アセット名: door) AstroObject型

## AstroCatalogクラス

存在するカテゴリーとアセットの一覧

- プロパティ
  - Categories: 存在するカテゴリー名の一覧
  - Assets: アセットごとのName、Category、Colors を持つ
    - Name: アセット名
    - Category: カテゴリー 例: Structure
    - Colors: 実装されているカラバリ番号一覧  例:00, 01
- 備考
  - Prefabのアセット名は Name + Color になる (例: door_01)
  

## AstroObjectクラス

AstroObject をシリアライズした json データ

- プロパティ
  - Name: アセット名
  - Category: 分類名
  - Tags: 絞り込み用の名前の一覧
  - Size、Center: 全体を囲む直方体の大きさと、原点のずれの補正値
  - Joints: AstroJoint の一覧
  - AssetID: 初期は空で生成時やカラー変更時に値変わる

## AstroJointクラス

他オブジェクトと接続するための取り付け口

- プロパティ
  - Position: 位置
  - Normal: 接続面の相手の向き
  - Size: 固定値

## AssetManagerクラス

