# 変更差分に基づくテスト観点（Stage5MidBoss・BomberFly・共通）

本ドキュメントは、今回の変更（BaseEnemy の `specificItem` 追加、BomberFly の新スクリプト適用とライト連動、Explosion プレハブの当たり・消滅パラメータ変更、GameManager のターゲットFPS設定等）に基づき、ユニット/結合/システムテストの観点を整理したものです。

## 変更箇所の概要と想定影響
- **BaseEnemy.cs**
  - `specificItem` を追加し、設定があれば `item` プロパティへ反映。
  - 影響: アイテムドロップ仕様、派生クラス全体の出現物に影響。
- **BomberFly.prefab / BomberFly.cs**
  - 新スクリプト（`BomberFly`）適用。移動/回頭ロジック、HP0時の赤点滅（加速）→爆発、ライト連動点滅を実装。
  - 影響: 追尾挙動、死亡演出、ライト負荷、サウンド。
- **Explosion/BomberFly_FX_explosion_air_03_a.prefab**
  - `SphereCollider.m_Radius` 2→2.5、`DeleteObject` 相当のパラメータ更新（`deleteTime:2`、`deleteColliderOnly:1`、`colliderDeleteTime:0.1`）。
  - 影響: 爆発ヒット範囲/時間、コライダー先行消滅挙動。
- **GameManager.prefab**
  - `m_targetFrameRate: 90` 追加。
  - 影響: FPS制御・端末依存挙動の差異。

---

## ユニットテストレベルでのテスト観点

- **listUnit_BaseEnemy_SpecificItem**
  - `specificItem` 設定時に `Start()` 経由で `item` に反映される。
  - 未設定時に `item` が既存仕様通り（null のまま or 他処理で設定）である。
- **listUnit_BomberFly_FlashSequence**
  - HP0→`enemyDie()` 呼出で 2 秒の点滅が開始され、点滅間隔が `initialFlashInterval`→`finalFlashInterval` に線形補間される。
  - 点滅終了時に元の色へ復帰し、`Explosion()` が呼ばれる。
  - `warningSound` 未設定でも例外が出ない／設定時 1 回のみ再生。
- **listUnit_BomberFly_LightSync**
  - `flashLight` 未設定時に例外なし。
  - 設定時、赤点滅 ON/OFF に同期して `flashLight.enabled` が ON/OFF 切替。
- **listUnit_BomberFly_MaterialCollection**
  - 子孫 `Renderer`/`SkinnedMeshRenderer` のマテリアルを重複なく収集し、`_BaseColor` / `_Color` の元色を保持。
- **listUnit_BomberFly_Movement**
  - 点滅中は移動停止。
  - 通常時はプレイヤーへ接近（X固定）、`stopDistance` 未満で停止。`Quaternion.LookRotation` 例外ガードが効いている。
- **listUnit_Explosion_Params**
  - コライダー半径 2.5 を想定したヒット範囲計算が正しく行える（スクリプト側が参照する場合）。
  - コライダーのみ 0.1 秒で無効化され、全体は 2 秒で消滅する前提の制御に問題がない（依存スクリプトがある場合）。
- **listUnit_GameManager_TargetFps**
  - `m_targetFrameRate=90` の設定読み込み・反映ロジック（該当メソッドがある場合）に不整合がない。

---

## 結合テストレベル（コンポーネント/オブジェクト間の相互処理）

### テストコードで確認すべき結合テスト

- **listIT_Code_BaseEnemy_DropItem**
  - `specificItem` 設定時、撃破で `ApearItem(item)` が `specificItem` を出現させる。
  - カメラ範囲外などの出現抑制条件が満たされない限り、Prefab が正しい座標に生成。
- **listIT_Code_BomberFly_Flash_Explode_Light**
  - HP0 シナリオで、(1)赤点滅→(2)ライト同期→(3)爆発の順序が保証される。
  - 点滅中に移動/攻撃が停止している。
- **listIT_Code_Explosion_HitWindow**
  - 爆発コライダーが 0.1 秒で無効になり、当たりがそこで終わる（当たり継続が起きない）。
  - ただしエフェクトは 2 秒まで残存（見た目と当たりの乖離が設計通り）。
- **listIT_Code_GameManager_Fps**
  - 起動時にフレームレートが 90 に設定される（プラットフォーム差・VSync 設定の影響があっても処理が例外なく流れる）。

### 実プレイで確認すべき結合テスト

- **listIT_Play_BomberFly_TrackingAndFacing**
  - BomberFly がプレイヤーへ接近し、常に自然な回頭で正対する。
  - `stopDistance` 前後での減速/停止の見た目が意図通り。
- **listIT_Play_BomberFly_Flash_Light_Explode**
  - HP0→赤点滅が徐々に高速化、ライト点滅も完全同期。終了と同時に爆発し、元色に戻る。
- **listIT_Play_Explosion_Radius**
  - 爆発の当たり範囲が広がったこと（半径 2.5）がプレイフィール的に適切か（当たり過ぎ/当たらなさ過ぎが無い）。
  - 当たりが早期（0.1s）に終わるため、多段ヒット等の副作用が無い。
- **listIT_Play_TargetFps**
  - 90FPS の指定が体感上問題ないか（過負荷端末でカクつき/発熱/バッテリー消費）。

---

## システムテストレベルでのテスト観点

- **listST_Performance_Stress**
  - BomberFly 複数同時出現（ライト多数点滅・サウンド併用）でフレームレートが目標値近傍を維持。
  - 爆発同時多発（大コライダー多数）で GC スパイク/ドロップが顕著でない。
- **listST_Compatibility_Rendering**
  - URP/Standard/カスタムシェーダ混在下で透明→不透明切替（MidBoss 側機構含む）が他エンティティへ副作用を与えない。
- **listST_RecoveryAndLifecycle**
  - 一時停止/再開、シーン再読込、ゲームオーバー遷移などライフサイクル操作でライト/マテリアル/当たりの残留が無い。
- **listST_AudioVisualQuality**
  - 警告音・爆発音の音量バランス、ライト点滅の眩惑度、点滅速度の体感品質が意図どおり。
  - 爆発演出（見た目2秒・当たり0.1秒）の乖離がプレイヤー体験として直感的。

---

## 備考（モック/補助）
- マテリアルはランタイム `Material` に対する `GetColor/SetColor`、`GetFloat/SetFloat` の実測で確認。
- ライトは `enabled` の履歴ロギングで点滅回数・タイミング検証。
- 爆発コライダーは当たりログ（衝突/Trigger）で 0.1s のみ発火することを確認。
- 追尾/回頭は位置・回転履歴サンプリングで単調性/安定性を確認。
