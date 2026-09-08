# UIRules.md

ユーザーがUIを実装するときのルール
※ライブラリを実装する場合はこのルールは未適用

## 対象

- 全ての.umxlファイル

## 絶対遵守事項

- XeroのUIコンポーネントだけを使用する
- UGUIを使用しない
    - VisualElment、Button Toggle等の標準UI要素を使わない
- ussファイルは使用しない

## UI要素命名規約

UIBuilder上でつける名前を指す

| コンポーネント名 | 名前 |
| UIView は view_xx | 
| UIButton は btn_xx | 
| UITextField は txf_xx | 
| UIDropdown は drrop_xx| 
| UICheckBox は cbx_xx | 
| UILabel は label_xx | 
| UITabView は tab_xx | 
| UITabPage は page_xx | 
| UIListView は list_xx | 
| UIHierarchyView は hie_xx| 
| UIModalView は modal_xx | 

※ xx は 2桁のナンバリングを使う

## 実装ルール

- UIListView は itemTemplate の設定を必須とする
    - ユーザーがUIBuilderで設定する運用
- UXMLはGUID形式を必須とし、/Assets/ 形式や相対パス形式を使わない


