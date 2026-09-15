# UIToolKit.md

## VisualElementの破棄について調査ログ

VisualElementはOnDestroyのような破棄タイミングを持っていない
VisualElementを継承した場合は親が破棄メソッドを呼ぶ必要がある

## UnityのThemeを自作したものに変更した件

標準UIのホバー色を無効にするため、Unityデフォルトテーマを使わず、Themeを自作した。
デフォルトテーマはUSSだけでは成立せず、内蔵されたUI画像とフォントが必要だった。UI画像はデフォルトテーマから抽出し、高解像度版を通常画像として使用した。内蔵フォントは独立して扱えなかったため、日本語対応のNoto Sans JPへ置き換えた。

これにより、標準UIの表示を維持したまま、スタイルをXero側で変更できるようになった。

## VisualElementの破棄とcallback登録状態に関する調査ログ

対象

- UIObjectではなくVisualElementそのものを対象にする。
- EventManagerの設計は、この調査の対象外とする。
- 調べるべき中心は、VisualElementがUnityから破棄されたと認識できる時点と、callback登録状態をどう管理するかである。

調査で整理した事実

- VisualElementのDetachFromPanelEventは、パネルとの接続解除を通知する。破棄通知としては扱えない。
- 同じVisualElementは階層から外した後に再追加できる。Detach後も同じインスタンスを再利用する経路がある。
- UIDocumentの有効無効では、以前の要素ツリーとは別のVisualElement群が生成される場合がある。
- RegisterCallbackの登録先はVisualElementである。登録状態は、同じVisualElementを再利用する限り管理対象として残る。
- UnregisterCallbackには登録時と同じdelegateが必要になる。登録時にdelegateを保持しないラムダを使うと、後からその登録だけを指定して解除できない。

不具合になる条件

- AttachToPanelEventなど、複数回起き得る処理の中で同じcallbackを毎回RegisterCallbackする。
- 同じVisualElementを再接続又は再利用したとき、登録済みcallbackに加えて同じ処理を再登録する。
- 長く残る親VisualElementに登録したcallbackが、削除済みの子VisualElement又は短命の処理を保持する。

現行Xeroの確認箇所

- UIListViewのInitializeは一度だけ実行されるため、セルのPointerUp callbackは重複登録しない構成になっている。
- UIExtensionのAddClickDownEventは呼び出すたびに新しいラムダをRegisterCallbackする。同じVisualElementに複数回呼ぶと、callbackが重複する可能性がある。delegateを保持していないため、個別解除はできない。
- DragManipulatorはRegisterCallbackとUnregisterCallbackを対にしている。

未検証

- Unity 6000.3.8f1で、同一VisualElementをDetachして再接続した場合に、登録済みcallbackが残ること。
- UIDocumentの有効無効時に、古いVisualElementのcallback登録状態がどの時点まで残るか。
- VisualElementがUnityから破棄済みと認識される専用の通知又は状態が存在するか。

次に必要な確認

- 同一VisualElementについて、初回接続、Detach、再接続、再登録後のcallback呼出し回数を人間がUnity上で確認する。
- UIDocumentの有効無効前後で、取得したVisualElementのインスタンス参照とcallback呼出し回数を人間がUnity上で確認する。
- 検証結果を基に、VisualElementのcallback登録を一意に管理する仕組みが必要かを判断する。

## Submit入力によるButton押下

- 発生した事象
    - ButtonをクリックしてDialogを表示した後にSpaceを押すと、フォーカスが残っていたButtonが再実行された
    - 再実行によって、表示中のDialogの上へ新しいDialogが生成された
- 原因
    - UI ToolkitのButtonは標準でフォーカス可能になっている
    - ButtonはNavigationSubmitEventを受け取るとクリックを実行する
    - マウスでButtonをクリックした後も、そのButtonへキーボードフォーカスが残る
    - InputManagerのSubmitにSpaceが割り当てられていたため、SpaceからNavigationSubmitEventが生成された
    - 画面全体をVisualElementで覆っても、背面要素のキーボードフォーカスは解除されない
- Xeroで必要な動作
    - Buttonはポインター操作だけで実行する
    - Space、Return、Enter、ゲームパッドからButtonを実行しない
- 対策
    - InputManagerにある2つのSubmit設定から入力割り当てを削除した
        - Return
        - Enter
        - Space
        - joystick button 0
    - Dialogや個別Buttonでは入力を遮断しない
        - NavigationSubmitEventへ変換された後は、元の入力がSpaceかEnterかをButton側で判別できないため
- 確認済み
    - Unity 6000.3.8f1で動作を確認した
    - Dialog表示中にSpaceを押しても、背面のButtonが再実行されないことを確認した
    - Return、Enter、ゲームパッドのSubmit操作からButtonが実行されないことを確認した

## TextFieldの入力制限によるカーソル位置の不整合

- 発生した事象
  - Numeric の UITextField へ全角文字を入力してEnterで確定した直後、次の文字を入力すると ArgumentOutOfRangeException が発生した
  - Unity内部の文字列挿入処理で、挿入位置が文字列の有効範囲を超えていた
- 正常時との違い
  - 半角数字の入力では、表示文字列とカーソル位置が同時に更新される
  - 全角文字の確定時は、入力制限によって表示文字列だけが空になり、Unity内部のカーソル位置と選択位置が全角文字の入力後の位置に残った
- 原因
  - ValueChanged の処理中に SetValueWithoutNotify で文字列を短くした際、カーソル位置と選択位置が新しい文字列長へ同期されていなかった
  - 次の入力で範囲外の位置へ文字を挿入しようとして例外が発生した
- 対策
  - 入力制限によって文字列を変更する前に、カーソル位置と選択位置を変更後の文字列長以内へ補正する
  - 文字列の変更後に SelectRange を使用し、補正した位置をUnity内部へ同期する

## TextFieldの文字数制限

- UITextField は Unity標準の TextField を継承している
- TextField が持つ max-length Attribute を UITextField でも使用できる
- max-length へ最大文字数を指定すると、キーボード入力と貼り付けの両方が指定文字数までに制限される
- デフォルト値は -1 で、文字数を制限しない
- UITextField 側へ同じ文字数制限処理を重複実装しない
- 入力種別を指定する input-type と併用できる
  - Numeric と max-length を併用すると、半角数字だけを指定文字数まで入力できる
