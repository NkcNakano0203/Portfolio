チーム制作作品（**プログラマー**6人）

**制作期間**：
**完成までの時間**：2023/4/10 ~ 2023/5/31（2ヶ月）
**バグ修正+リファクタリング**：2023/6/1 ~ 2024/3/1（6ヶ月程度、不定期制作）
**開発環境**：Unity(**2022.3.18f1**)、C#、DSPAcion(SE制作ツール)、GitHub、Sourcetree
**ライブラリ**：UniTask,UniRx,DOTween,CRIWARE

**操作方法**：
**キーボード**：
	**A,D**：移動
	**Enter**：決定
	**Space**：ジャンプ
	**ESC**：メニュー
	
**Xboxコントローラー**：
	**左スティック**：移動
	**Aボタン**：ジャンプ
	**STARTボタン**：メニュー
	
**DualSenseコントローラー**：
	**左スティック**：移動
	**✕ボタン**：ジャンプ
	**STARTボタン**：メニュー

**ゲーム概要**：
レーザーを曲げて、避けて進む、3D横スクロールパズルアクションゲームです。
パズルアクションゲームを作りたいとなった時に、レーザーを動かすと視覚的にも派手で解けた時に綺麗じゃないか と言う案から始まりました。
日本ゲーム大賞2023アマチュア部門とゲームクリエイター甲子園2023に向けて制作しました。
ゲームクリエイターズギルドEXPO2023ではB.B.スタジオ賞に選ばれました。
第12回全国専門学校ゲームコンペティション プレイアブル部門ではファイナリストに選ばれました。

**こだわった点**：
**シーン管理**
加算シーンを用いてステージSceneと管理Sceneを分けることで、
同時に作業できるようにしたり、共通するものをまとめることができました。
![[Pasted image 20240627010812.png]]

**UIの実装**
UIをMVPパターンを参考に実装したので、テストや差し替えがしやすくなった
**Presenter**：ModelとViewの橋渡し役であり、これ自体は特に処理はしない
```C#
public class PausePanelPresenter : MonoBehaviour
{
    [SerializeField]
    PausePanelModel model;
    [SerializeField]
    PausePanelView view;
    void Start()
    {
        model.PanelToggle
            .Subscribe(view.ChangeActive)
            .AddTo(gameObject);

        view.BackButtonCliecked
            .Subscribe(x => model.InputButton())
            .AddTo(gameObject);
    }
}
```
**Model**：アクティブ情報などのデータを持つ役
```C#
public class PausePanelModel : MonoBehaviour
{
    private ReactiveProperty<bool> panelToggle = new(false);
    public IReadOnlyReactiveProperty<bool> PanelToggle => panelToggle;
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    public void InputButton()
    {
        panelToggle.Value = !panelToggle.Value;
        eventSystem.SetSelectedGameObject(firstSelectButton.gameObject);
    }
}
```
**View**：Buttonなどの画面上にあり、ユーザーのアクションを受ける役
```C#
public class PausePanelView : MonoBehaviour
{
    private Subject<Unit> backButtonSubject = new Subject<Unit>();
    public IObservable<Unit> BackButtonCliecked => backButtonSubject;
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    public void ChangeActive(bool active)
    {
        if (active)
        {
            Enable().Forget();
        }
        else
        {
            Disable().Forget();
        }
    }
    private void backButtonClick()
    {
        backButtonSubject.OnNext(default);
    }
}
```

**セーブデータの暗号化**
排他的論理和を用いたセーブデータの暗号化に挑戦しました。

```C#
/// <summary>
/// 排他的論理和
/// dataとkeyのbyte[]を一文字ずつ比較していく
/// </summary>
public static byte[] Xor(byte[] dataBytes, byte[] keyBytes)
{
	int i, j = 0;

	for (i = 0; i < dataBytes.Length; i++)
	{
		j = (j < keyBytes.Length) switch
		{
			true => j + 1,
			// keyの文字数が足りない時は１を置く
			false => 1
		};

		dataBytes[i] = (byte)(dataBytes[i] ^ keyBytes[j - 1]);
	}
	return dataBytes;
}
```

暗号化するとデータの変更が難しくなるので、
開発中の対策としてScriptableObjectのインスペクタを拡張して操作しやすくしました。
![[Pasted image 20240627015113.png]]

**課題**：
Scene間のデータのやり取りを何を使って行うか

**解決法**：
ScriptableObjectだと削除されずに残ってしまうので、
UnityのScene読み込み時のイベントにメソッド登録して参照が残らないようにして、
参照関係を綺麗にした。
`GameManager.cs：55行目`
```C#
// シーン読み込み時のイベントに登録
SceneManager.sceneLoaded += SetEvent;
string stageName = $"Stage_{currentStage.Number}";
// 加算シーンとしてシーンをロード
await SceneManager.LoadSceneAsync(stageName, LoadSceneMode.Additive);
playerTransform.position = startPos;
restartPosition.currentRestartPos = startPos;
```
新しいシーンから必要なクラスを取得して情報のやり取りをする
`GamaeManager.cs：115行目`
```C#
async public void SetEvent(Scene loadedScene, LoadSceneMode mode)
{
	foreach (var item in loadedScene.GetRootGameObjects())
	{
		// 読み込んだシーンから必要なクラスを取得
		if (!item.TryGetComponent(out Stage nextStage)) continue;

		// リスタートイベントに再読み込みメソッドを登録
		nextStage.SetRestartEvent(() => ReLoad().Forget());
~~~~~~~~~~~~~~~~~~~~~~
```
