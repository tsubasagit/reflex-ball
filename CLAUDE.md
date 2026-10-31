# CLAUDE.md — reflex-ball（いけの ボール タッチ！）

## 構成
- `index.html` 1 枚で完結（HTML / CSS / JS / GLSL をすべて同梱、依存ライブラリなし、ビルド不要）
- `SERVICE_SPEC.md` に仕様。機能を変えたらこちらも更新する

## 動かし方
ブラウザで `index.html` を開くだけ。ローカルサーバーを立てるなら:

```bash
python -m http.server 8997
```


## 描画の層（下から）
1. `#gl` WebGL キャンバス。全画面三角形 1 枚に、`#fs` のフラグメントシェーダーで池・蓮の葉・ボールをまとめて描く
2. `#ball` DOM。WebGL が使えるときは `visibility:hidden` で当たり判定の位置合わせだけに使う（WebGL 不可のときの代わりのボール）
3. `#spray` 2D キャンバス。カエル（`drawFrog`）、金のボールのきらめき（`drawSparkles`）、空中の水滴と影
4. `.msg` `.banner` `.countdown` `.float-pt`（触った所の点数）の文字、`.stage::after` の状態色（`rainy` `fever` `count-on`）
5. `#result` 結果カード（池の上に重ねる。ここでの pointerdown は池の操作にしない）

## 水面の波
- JS 側で高さの格子（横 210 マス、幅 520px 未満は 150 マス）に波動方程式を解く。`simStep()` を 1 秒 150 回
- 毎フレーム `encodeSim()` で RGBA8 テクスチャに詰めて送る。R,G = 傾き、B = 曲率（光の集まり）、A = 高さ
- 波を起こすのは `addDrop(x, y, 半径px, 強さ)`。正で盛り上げ、負でへこませる。`plow()` は前をへこませ後ろを戻すことで、進む物の航跡を作る

## シェーダーの座標
- `p` は縦の長さを 1 とした座標で、y は上向き。横は `uAspect` まで
- JS から渡すときは `x/ch` と `1 - y/ch` に直す（`renderGL()` を参照）
- 光は 2 本。`Ls` は影・ボール・葉の立体感（左上の太陽）、`Lg` は水面のきらめき用でほぼ真上

## はまりどころ
- 状態名はそのままステージの class になる。`idle` `count` `go` `done` `has-gl` `count-on` `rainy` `fever` と同じ名前の class を単独で CSS に書かない。カウントダウンの部品は `.countdown`（`.count` にすると池が消えた）
- ステージに付ける「WebGL 使用中」の目印は `has-gl`。キャンバスの class `.gl`（`position:absolute; inset:0; pointer-events:none`）と同じ名前にすると、池の枠が画面全体に広がってスタートボタンを覆い、タッチも素通りする（2026-10-31 に実際に起きた）
- 当たり判定は `touchesBall()`。今の位置に加え、直前 120ms（`HIT_LAG`）にいた位置も当たりにする。指が触れてから反応するまでの遅れの分。跳ねた直後は `trail` を空にする
- 乱数関数 `hash()` は入力を `mod 289` してから混ぜる。大きな数を掛ける型の hash だと、隣のマス目で計算順が変わって値がずれ、画面に格子状の線が出る
- 光の網 `caustic()` は `mod` で繰り返さない。繰り返すと継ぎ目が直線で見える
- ステージの幅が 0 のとき（非表示ペインなど）は `resize()` を飛ばし、`loop()` 側で大きさが決まった時点で作り直す
- 動作確認で内蔵ブラウザのペインが隠れていると rAF が止まる。ページ内で `window.requestAnimationFrame` を setTimeout に差し替えると回り出すが、タイマーも間引かれるのでボールの速さや航跡の濃さはその画面では判断できない

## 進行の実装メモ
- 状態は `idle | count | go | done` の 4 つ。`setState()` が `.stage` の class も切り替える（WebGL 使用時は `has-gl` を付け足す）
- 流れ: `insertCoin()` → `startGame()`（クレジット -1、`planEvents()` でイベントのくじ）→ `countdown()` → `go()` → 毎フレーム `gameTick()` → `endGame()`
- ゲーム内の時間 `timeLeft` は `loop()` の dt で減らす。setTimeout を使わないので、タブが裏に回るとゲームも止まる
- イベントは `events`（経過秒 `at` と種類）を開始時に作り、`runEvents()` が時刻になったものから実行する
- 得点は `hitBall()`（ボール）と `catchFrog()`（カエル）。倍率は `mult()`（コンボ数から）、ラストスパート中はさらに 2 倍
- 入力の優先順は ボール → カエル → 葉 → 水面（ミス）。`onPointer()` を参照
- 調整する値は先頭の定数（`GAME_TIME` `FEVER_TIME` `GOLD_TIME`）と、`planEvents()`・`hitBall()`・`TITLES`
- 効果音は `sfx(name)`。WebAudio で合成するので音声ファイルはない。最初の音はボタン操作のときに鳴らす（ブラウザの自動再生制限）
- ランキングは `RANK_KEY`。名前は `esc()` で無害化してから表示する
- 動作確認で全体を通すときは、ページ内でボールの DOM（`#ball`、表示は隠しているが位置は動く）の中心に pointerdown を送り続けると 30 秒を最後まで遊べる
