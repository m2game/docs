# Xero.md

## ユーザーダイアログの実装方法

- 目的
  - 利用側がUI Toolkitのクラスを参照せずに、独自の内容を持つダイアログを作成する
- 前提
  - コンテンツはユーザー側がUI BuilderでXeroのUIコンポーネントを使って作成する
  - 利用側では using UnityEngine.UIElements; を書かない
  - 利用側ではUI Toolkitのクラスを参照しない
- 実装手順
  - コンテンツのUXMLを作成し、取得する各要素に名前を設定する
  - 利用側のスクリプトにSerializeField付きのUITemplateフィールドを定義する
  - InspectorでUITemplateフィールドに作成したUXMLを指定する
    - XeroのUITemplateDrawerが、内部のアセット参照を直接指定できる欄を表示する
  - UIDialogView.CreateにUITemplateを渡してダイアログを生成する
  - FindChildでXeroのUIコンポーネントを取得し、初期値や応答ボタンのイベントを設定する
  - Showでダイアログを表示する
- コード例
  - UXMLに txf_01 と txf_02 という名前のUITextFieldを配置する

```csharp
using UnityEngine;
using Xero;

public class DialogSample : MonoBehaviour
{
    [SerializeField] UITemplate _dialogContentTemplate;

    public void ShowDialog()
    {
        var dialog = UIDialogView.Create(_dialogContentTemplate);
        var horizontalField = dialog.FindChild<UITextField>("txf_01");
        var verticalField = dialog.FindChild<UITextField>("txf_02");

        dialog.SetResponseButtons("作成", () =>
        {
            Debug.Log($"横マス={horizontalField.Text}, 縦マス={verticalField.Text}");
        });
        dialog.SetCloseButton(() => { });
        dialog.Show();
    }
}
```

- 内部の仕組み
  - UITemplateはVisualTreeAssetを内部に保持するSerializableクラスとする
  - UIDialogView.Createが内部のアセット参照をContextScreenへ渡し、共通のダイアログ枠とコンテンツを生成する
  - VisualTreeAssetの空継承は使用しない
    - 通常のUXMLアセットはVisualTreeAssetとして生成され、派生型のフィールドには割り当てられないため
- 注意点
  - 表示前にUITemplateのUXML参照を設定する
  - FindChildに指定する名前と型をUXMLの要素に一致させる
  - 応答ボタンと閉じるボタンは、設定したActionを実行した後にダイアログを閉じる
  - 既存のVisualTreeAssetフィールドをUITemplateへ変更する場合は、保存済みの参照をUITemplate内部のアセット参照へ移行する
  - TODO: Unity上でコンパイル、InspectorでのUXML指定、ダイアログの表示と入力・応答を人間が確認する

## ユーザーダイアログのバリデート実装方法

- テキストフィールドなどの値が不正な場合
  - ダイアログを閉じないようにします
- ValidateEventに判定処理を設定してください
  - 正常ならtrueを返してください
  - 不正ならfalseを返し、必要に応じてエラー等を表示してください

```csharp
var dialog = UIDialogView.Create(_dialogContentTemplate);
var horizontalField = dialog.FindChild<UITextField>("txf_01");
var verticalField = dialog.FindChild<UITextField>("txf_02");

dialog.ValidateEvent = () =>
{
    if (string.IsNullOrEmpty(horizontalField.Text) ||
        string.IsNullOrEmpty(verticalField.Text))
    {
        Debug.LogError("入力されていない項目があります");
        return false;
    }

    return true;
};
dialog.SetResponseButtons("作成", () =>
{
    Debug.Log($"横マス={horizontalField.Text}, 縦マス={verticalField.Text}");
});
dialog.Show();
```

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

## コンテキストメニューの実装方法

- 目的
  - 利用側がUIContextViewの破棄状態を管理せずにコンテキストメニューを表示する
  - 閉じたUIContextViewを保持して再利用する経路を作らない
- 前提
  - ContextScreenを画面へ配置して初期化しておく
  - UIContextViewはメニューを閉じたときにXero内部で破棄される
- 実装手順
  - メニューを表示する処理内でUIContextView.Createを呼ぶ
  - 生成したUIContextViewへ項目を追加する
  - SetClickEventで選択時の処理を設定する
  - Showで表示する
  - UIContextViewをフィールドへ保存しない
- コード例

    void ShowContextMenu(UIButton button)
    {
        var contextView = UIContextView.Create();
        contextView.Add(MenuItem.Edit, "編集", true);
        contextView.Add(MenuItem.Delete, "削除", true);
        contextView.SetClickEvent(id =>
        {
            Debug.Log($"選択: {id}");
        });
        contextView.Show(
            UIHelper.GetPanelPosition(button, UIAnchorPosition.BottomLeft));
    }

- 注意点
  - UIContextViewは表示のたびに生成する
  - 閉じたUIContextViewは再利用しない
  - UIContextViewをフィールドへ保持しない
  - ContextScreenからUIContextViewを取得しない

## 階層型コンテキストメニューの実装方法

- 目的
  - 通常の項目と階層を持つ項目を同じUIContextViewへ追加する
  - 親項目から子項目を開き、末端項目だけを選択対象にする
- 前提
  - 通常のコンテキストメニューと同様に、ContextScreenを画面へ配置して初期化しておく
  - UIContextViewとContextMenuItemをフィールドへ保持しない
- 実装手順
  - UIContextView.Createでコンテキストメニューを生成する
  - 通常項目はUIContextView.Addで追加する
  - 親項目はUIContextView.AddGroupで追加する
    - 戻り値のContextMenuItemへAddを呼び、子項目を追加する
    - 戻り値のContextMenuItemへAddGroupを呼び、さらに下の階層を追加する
  - SetClickEventで末端項目が選択されたときの処理を設定する
  - Showで表示する
- 通常メニューのコード例

    void ShowContext1(UIButton button)
    {
        var contextView = UIContextView.Create();
        contextView.Add(MenuItem.Item1, "Item1", true);
        contextView.Add(MenuItem.Item2, "Item2", true);
        contextView.AddDivider();
        contextView.Add(MenuItem.Item3, "Item3", true);
        contextView.Add(MenuItem.Item4, "Item4", false);
        contextView.AddDivider();
        contextView.Add(MenuItem.Item5, "Item5", true);
        contextView.SetClickEvent(id =>
        {
            Debug.Log($"選択: {id}");
        });
        contextView.Show(
            UIHelper.GetPanelPosition(button, UIAnchorPosition.BottomLeft));
    }

- 2階層メニューのコード例

    void ShowContext2(UIButton button)
    {
        var contextView = UIContextView.Create();
        contextView.Add(MenuItem.Item1, "Item1", true);

        var item2 = contextView.AddGroup("Item2", true);
        item2.Add(MenuItem.Item3, "Item3", true);
        item2.Add(MenuItem.Item4, "Item4", true);
        item2.Add(MenuItem.Item5, "Item5", true);

        contextView.SetClickEvent(id =>
        {
            Debug.Log($"選択: {id}");
        });
        contextView.Show(
            UIHelper.GetPanelPosition(button, UIAnchorPosition.BottomLeft));
    }

- 動作
  - 子項目を持つ親項目へマウスを合わせると、右側へ次の階層が表示される
  - 子項目を持たない項目へマウスを合わせると、不要な下位階層が閉じる
  - 親項目は選択イベントを発行しない
  - 末端項目を選択すると、選択イベントを発行して全階層を閉じる
  - メニュー外を押すと、選択イベントを発行せずに全階層を閉じる
- 注意点
  - AddGroupの戻り値は、そのメニューを構築している間だけ使用する
  - 閉じた後のUIContextViewとContextMenuItemを再利用しない
  - 階層の深さは固定されていないため、ContextMenuItemへAddGroupを繰り返して追加できる

## CacheManager の使い方

- 目的
  - メモリ内に保持した値を再利用し、未登録キーによる KeyNotFoundException を避ける
- 前提
  - CacheManager が保持するのはメモリ内の値だけ
  - 対応する型は string、Texture2D、GameObject
  - 値は型とキーの組で管理される
  - Get は、指定した型とキーの組が存在しない場合に KeyNotFoundException を投げる
- 実装手順
  - 読み込み前に Exist で同じ型とキーの組を確認する
  - 存在する場合だけ Get を呼ぶ
  - 存在しない場合は元データを読み込み、Set で登録する
- コード例

    using UnityEngine;
    using Xero;

    public class PrefabCacheSample : MonoBehaviour
    {
        const string PrefabKey = "door.prefab";

        [SerializeField] GameObject _sourcePrefab;

        public GameObject GetPrefab()
        {
            if (CacheManager.Instance.Exist<GameObject>(PrefabKey))
            {
                return CacheManager.Instance.Get<GameObject>(PrefabKey);
            }

            CacheManager.Instance.Set(PrefabKey, _sourcePrefab);
            return _sourcePrefab;
        }
    }

- 注意点
  - Set には null を渡せない。キーには null、空文字、空白だけの文字列を使えない
  - Set 後も、Delete、Clear、件数上限による追い出しがあればキーは存在しなくなる
  - Get は使用順を更新する。件数上限に達したときは、最後に使用してから最も時間が経ったキーが追い出される
  - SetMaxCount は型ごとに設定する。0 を指定すると新しい値は保持されないため、Set の直後でも Get は失敗する
  - Unlimited を指定すると件数上限を解除する
  - キャッシュはアプリをまたいで保存されない
  - Set で渡した GameObject と Texture2D は、Delete や Clear のときに CacheManager が破棄しない
