# CodingRules.md

スクリプトファイルを編集するときルール

## 対象

- 全ての.csファイル

## 絶対遵守事項

- Xeroライラブラリを優先で使用する
- using UnityEngine.UIElements; は禁止とする
    - Xeroライブラリ実装のためであれば使用を許可

## 命名規則

- クラス名:  PascalCase
- メソッド名: PascalCase  動詞から始める
- プロパティ名: ascalCase
- フィールド名: _camelCase
    - 例: _currentTime;
- 定数名: PascalCase
- bool型: is has can が接頭語が必ずつくようにする
    - 例: _isOpen; IsOpen(); IsOpen { get;}

## 実装/改修/修正におけるルール

- UI 要素取得で Q は直接使わずXeroの拡張メソッドを使う
- 新規実装でnullチェックはしない
    - 既存コードにあるのは人間がメンテナンスしているので問題なし
- 既存の this. は消さないこと
- 既存の using は消さないこと
- プログラムでデザインを実装しない
    - コンストラクタでレイアウトや見た目用の style を追加しない
- SharedPreferencesのキー文字列はconstにする
- マジックナンバーは使用禁止、定数化して名前を付ける
- 露出が必要なフィールドは public にせず SerializeField を使う
- this. は書かない
    - 例外として拡張メソッドやインスタンスメンバー解決する場合のみ使用可
- ref は使わない

## コーディングルール

- セッタープロパティは原則使わないこと。
    - 必ずSetXXXという関数を使って値を渡す
    - 例外としてはEntityやScriptableObjectのようなデータのみを持つクラスはセッター、ゲッターを使うようにする
    - Action を受け取るイベント設定は、プロパティでセットする仕様としてよい

- 取得処理はゲッタープロパティを使い、関数は使わない 

- FindViewで取得するものは抽象クラスのを優先すること
    - 例) UIImageButton → UIButton
    - 継承クラスにしかない機能使う場合は例外とする