# Xero.md

## ユーザーダイアログの実装方法

- コンテンツはユーザー側で作成
- VisualTreeAssetとして渡す
- UIDialogView.Createで生成
- FindChildで取得しイベント設定

## ライブラリの初期画像をランタイムで補う設計

- 目的
  - ユーザーが画像を設定していない箇所だけに初期画像を補う
  - Image などの画像属性と、背景スタイルによる画像指定の両方を尊重する
  - 配布に必要な画像とコードを Assets/Xero 内に収める
- 前提
  - Unity 6000.3 系の AddAdditionalPathToStreamingAssets を使用する
  - 現在のファイル読み込み方式は macOS など、通常のファイルAPIで StreamingAssets を読み込める環境を対象とする
- 実装手順
  - 初期画像を Assets/Xero/StreamingAssets/UI に配置する
  - Editor フォルダ内の BuildPlayerProcessor で、画像ファイルを個別にビルドへ登録する
    - Assets/Xero/StreamingAssets は、名前だけでは自動コピーの対象にならない
    - meta ファイルは登録しない
  - エディタでは Application.dataPath を基準にライブラリ内の画像を読む
  - アプリでは Application.streamingAssetsPath を基準に登録先の画像を読む
  - PNGを Texture2D.LoadImage で読み込み、共通の初期画像としてキャッシュする
  - UXMLの属性とスタイルが反映された後に、未設定の箇所だけを補う
    - コンストラクタや属性のゲッターで初期画像を無条件に読み込まない
    - UIImageButton はレイアウト確定後に、画像属性、インライン背景、解決済み背景の有無を確認する
    - UICheckBox はユーザー指定画像と初期画像を別に保持し、指定画像を優先する
- ビルドへの登録例

```csharp
using System.IO;
using UnityEngine;

namespace Xero.Editor
{
    public class UIInitialImagesBuild : UnityEditor.Build.BuildPlayerProcessor
    {
        const string SourceDirectory = "Xero/StreamingAssets/UI";
        const string DestinationDirectory = "Xero/UI";
        static readonly string[] ImageNames =
        {
            "icon_square_fill.png",
            "icon_square.png",
            "icon_check.png"
        };

        public override void PrepareForBuild(UnityEditor.Build.BuildPlayerContext context)
        {
            foreach (string name in ImageNames)
            {
                string source = Path.Combine(Application.dataPath, SourceDirectory, name);
                context.AddAdditionalPathToStreamingAssets(source, DestinationDirectory + "/" + name);
            }
        }
    }
}
```

- UIImageButton の未設定判定例
  - 次の処理を GeometryChangedEvent から呼び出す
  - UIInitialImages.ButtonImage は、初期画像を読み込みキャッシュする共通処理とする

```csharp
void ApplyInitialImage()
{
    if (!_isAttached || _image) return;
    if (!style.backgroundImage.value.Equals(default(Background))) return;
    if (!resolvedStyle.backgroundImage.Equals(default(Background))) return;

    _initialImage = UIInitialImages.ButtonImage;
    style.backgroundImage = _initialImage;
    _hasAppliedImage = true;
}
```

- 注意点
  - 初期画像をUSSに追加する方式は、このプロジェクトでは使用しない
  - Image 属性が空でも、背景スタイルに画像があれば未設定とは判定しない
  - ユーザー指定画像を補完用画像で上書きしない
  - Editor フォルダ外では using UnityEditor を使用しない
  - Editor フォルダ内のビルド処理を、この目的で UNITY_EDITOR に囲まない
  - Android と Web では通常のファイルAPIで読み込めないため、対応時は読み込み方式を別途検討する
  - TODO: 最新の補完処理について、macOSアプリで初期画像、属性指定画像、背景スタイル指定画像、指定解除後の表示を人間が確認する
