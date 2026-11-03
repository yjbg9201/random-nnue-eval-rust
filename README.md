# NNUE学習初期値ガチャ

`bias = 0, weight = {rand}` な初期値ファイルを適当に生成します。

## example

- make random eval HalfKP

```
cargo run --release --bin halfkp
```

- make random eval HalfKP (only affine layer)

```
cargo run --release --bin halfkp_affine from_halfkp_eval_dir/nn.bin
```

- make random eval HalfKPE9

```
cargo run --release --bin halfkpe9
```

- make random eval HalfKPE9 (only affine layer)

```
cargo run --release --bin halfkpe9_affine from_halfkpe9_eval_dir/nn.bin
```

- convert HalfKP to HalfKPE9

```
cargo run --release --bin convto_hkpe9 from_halfkp_eval_dir/nn.bin to_halfkpe9_eval_dir/nn.bin
```

- convert HalfKPE9 to HalfKP

```
cargo run --release --bin conv_hkp9_to_hkp from_halfkpe9_eval_dir/nn.bin to_halfkp_eval_dir/nn.bin
```

- playout filter

```
cargo run --release --bin psfen_filter from_genkifu/ to_genkifu/
```

## ソースファイル構成

```
.
├── Cargo.toml / Cargo.lock   # クレート定義（依存: rand, rand_distr, separator）
├── LICENSE                   # MIT ライセンス
└── src
    ├── lib.rs                # ライブラリのルート（eval_value モジュールを公開）
    ├── eval_value.rs         # NNUE 評価関数ファイルの共通処理
    └── bin/                  # 実行ファイル（cargo run --bin <名前>）
        ├── halfkp.rs
        ├── halfkp_affine.rs
        ├── halfkpe9.rs
        ├── halfkpe9_affine.rs
        ├── convto_hkpe9.rs
        ├── conv_hkp9_to_hkp.rs
        ├── analyze_hkp.rs
        ├── analyze_hkpe9.rs
        ├── psfen_filter.rs
        └── psfen_shuffle.rs
```

### ライブラリ (`src/`)

| ファイル | 役割 |
| --- | --- |
| `lib.rs` | クレート `random_nnue_eval` のルート。`eval_value` モジュールを公開するだけ。 |
| `eval_value.rs` | NNUE 評価関数 (`nn.bin`) のデータ構造と入出力をまとめたモジュール。<br>・`FeatureLayer`（入力特徴量層, i8）/ `AffineLayer`（全結合層, bias i16・weight i8）<br>・トレイト `EvalValueNnue`: ヘッダ・ハッシュ・各層サイズの定義、`zero()`（全0生成）、`clear_weight()`（正規分布の乱数で重みを再生成し bias を0にする。標準偏差に `NAN` を渡した層は元の値を維持）、`load()` / `save()`（ファイル読み書きとヘッダ検証）、`analyze()`（bias の最小・最大と重みの分布を表示）<br>・`EvalValueNnueHalfKP`（特徴量 125,388 × 512）と `EvalValueNnueHalfKPE9`（HalfKP の 9 倍, 125,388 × 9 × 512）の実装<br>・相互変換 `from_hkp()`（HalfKP の重みを 9 回複製）/ `from_hkpe9()`（先頭の HalfKP 分だけ切り出し） |

### 実行ファイル (`src/bin/`)

| ファイル | 役割 | 引数 |
| --- | --- | --- |
| `halfkp.rs` | 乱数初期化した HalfKP 評価関数を 10 個生成し `eval_halfkp/000〜009/nn.bin` に保存 | なし |
| `halfkp_affine.rs` | 既存の HalfKP 評価関数を読み込み、特徴量層はそのままで全結合層だけ乱数で再初期化して 10 個保存 | 入力 `nn.bin` |
| `halfkpe9.rs` | 乱数初期化した HalfKPE9 評価関数を 10 個生成し `eval_halfkpe9/000〜009/nn.bin` に保存 | なし |
| `halfkpe9_affine.rs` | HalfKPE9 版の `halfkp_affine` | 入力 `nn.bin` |
| `convto_hkpe9.rs` | HalfKP → HalfKPE9 へ変換 | 入力 `nn.bin`, 出力 `nn.bin` |
| `conv_hkp9_to_hkp.rs` | HalfKPE9 → HalfKP へ変換（バイナリ名は `conv_hkp9_to_hkp`） | 入力 `nn.bin`, 出力 `nn.bin` |
| `analyze_hkp.rs` | HalfKP 評価関数を読み込んで統計情報を表示 | 入力 `nn.bin` |
| `analyze_hkpe9.rs` | HalfKPE9 評価関数を読み込んで統計情報を表示 | 入力 `nn.bin` |
| `psfen_filter.rs` | 教師局面 (PackedSfenValue, 40 バイト/局面) の `.bin` ファイル群を加工して出力ディレクトリへ書き出す。入玉（玉の段）に応じた評価値加点、水匠方式の「反省」部分の評価値補正、勝敗と評価値の符号が食い違う局面の除外を行う | 入力ディレクトリ, 出力ディレクトリ |
| `psfen_shuffle.rs` | ディレクトリ内の教師局面 `.bin` をすべて読み込み、シャッフルして 1 ファイルにまとめる | 入力ディレクトリ, 出力ファイル |

> 乱数の標準偏差は、特徴量層 15、全結合層 1〜3 層目がそれぞれ 7 / 20 / 50 です（`*_affine` は特徴量層を維持）。
