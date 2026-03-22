# Ollama gfx900 スターターキット

<p align="center">
  <img src="assets/img/logo.webp" alt="Ollama gfx900 Starter Kit" width="200"/>
</p>

**AMD Vega / gfx900 / Radeon Instinct MI25** 向けに [Ollama](https://ollama.com) を動かすためのビルド済みバックエンドライブラリです。

上流の Ollama は gfx900 をサポートしていません。このキットは MI25 上での推論を可能にする HIP/ROCm バックエンドのコンパイル済みライブラリを提供します。

> **対象範囲:** Linux + ROCm のみ。上流の Windows HIP ビルドは gfx900 を除外しており、ここでもサポートしません。

[English](README.md) | 日本語

---

## 収録内容

```
build-gfx900/
└── lib/
    └── ollama/
        ├── libggml-base.so(.0/.0.0.0)     # GGML ベース
        ├── libggml-cpu-*.so                # CPU バリアント (haswell / sandybridge / sse42 / ...)
        └── libggml-hip.so                  # gfx900 向けにコンパイルされた HIP/ROCm バックエンド
```

これらのライブラリは gfx900 パッチ済みの `ollama` バイナリと組み合わせて使います（[バイナリの入手](#バイナリの入手) を参照）。

---

## 動作要件

| コンポーネント | 要件 |
|----------------|------|
| GPU | AMD Radeon Instinct MI25 (gfx900) |
| OS | Linux (Ubuntu 22.04 / 24.04 推奨) |
| ROCm | 6.1 以降 (`amdgpu-install`) |
| glibc | 2.35 以降 (Ubuntu 22.04 ベースライン) |

GPU が認識されているか確認:

```bash
rocminfo | grep -E "Name|gfx"
# 期待値: "gfx900" を含む行が表示される
```

---

## バイナリの入手

`ollama` バイナリはここには同梱されていません。以下いずれかの方法で入手してください。

**方法 A — ビルド済みバイナリをダウンロード（推奨）**

このリポジトリの [Releases](../../releases) ページから `ollama` をダウンロードし、`build-gfx900/` と同じディレクトリに置きます。

**方法 B — ソースからビルド**

```bash
git clone https://github.com/<your-org>/ollama-src
cd ollama-src
git checkout vega-int8-probe

# gfx900 向け HIP バックエンドをビルド
cmake -B build-gfx900 \
    -DAMDGPU_TARGETS=gfx900 \
    -DCMAKE_BUILD_TYPE=Release
cmake --build build-gfx900 --parallel $(nproc)

# ollama バイナリをビルド
go build -o ollama .
```

---

## セットアップ

### 1. スターターキットを配置

```bash
# 配置例（任意のディレクトリで構いません）
~/ollama-gfx900/
├── ollama                  # パッチ済みバイナリ
└── build-gfx900/
    └── lib/
        └── ollama/
            ├── libggml-base.so*
            ├── libggml-cpu-*.so
            └── libggml-hip.so
```

ランタイムはバイナリのディレクトリとカレントディレクトリの両方を基準に `build-gfx900/lib/ollama/` を探します。追加の設定は不要です。

### 2.（オプション）MI25 のみに固定

複数の GPU が搭載されている場合、Ollama を MI25 に固定できます:

```bash
# MI25 の UUID を取得
rocminfo | grep -A5 "gfx900" | grep "Uuid"

# 起動前に設定
export ROCR_VISIBLE_DEVICES=<UUID>
```

### 3. Ollama を起動

```bash
cd ~/ollama-gfx900
./ollama serve
```

ログで gfx900 の検出を確認:

```
level=INFO msg="forcing num_parallel=1 for ROCm gfx900" compute=gfx900
```

---

## gfx900 でのランタイム動作

gfx900 デバイスが検出されると、以下の設定が自動的に適用されます。環境変数の手動設定は不要です。

| 設定 | 値 | 理由 |
|------|----|------|
| Flash Attention | 無効 | gfx900 で非対応 |
| `num_parallel` | 1（強制） | gfx900 での安定性確保 |
| `num_ctx` デフォルト | 4096（MI25 16 GB 典型値） | VRAM ベースのデフォルト、最大 8192 |

ログに `FlashAttention:Auto` と表示されますが、これは期待通りの動作です。`OLLAMA_FLASH_ATTENTION` の値に関わらず Flash Attention は使用されません。

---

## モデルの取得と実行

```bash
./ollama pull tinyllama
./ollama run tinyllama
```

コンテキスト長が重要なワークロードでは `num_ctx` を明示的に指定してください（MI25 での最大値は 8192）:

```bash
./ollama run --num-ctx 4096 tinyllama
```

---

## rocBLAS / Tensile カーネル

システムの ROCm パッケージには gfx900 固有の Tensile カーネルが含まれていない場合があり、GEMM 性能に影響することがあります。性能低下や rocBLAS 初期化エラーが発生した場合は、[ROCm-MI25-build](https://github.com/AETS-MAGI/ROCm-MI25-build) のスクリプトでローカルカーネルをビルドしてください:

```bash
./build-rocblas-gfx900.sh
export ROCBLAS_TENSILE_LIBPATH=/path/to/local/rocblas/library
```

---

## トラブルシューティング

**Ollama が CPU にフォールバックする**

`rocminfo` に `gfx900` が表示されること、および `ROCR_VISIBLE_DEVICES` が無効な値に設定されていないことを確認してください。

**`libggml-hip.so: cannot open shared object`**

`build-gfx900/lib/ollama/` がバイナリと同じディレクトリツリーにあることを確認してください。ランタイムはバイナリの場所とカレントディレクトリを基準に探索します。

**rocBLAS 初期化エラー**

[ROCm-MI25-build](https://github.com/AETS-MAGI/ROCm-MI25-build) で gfx900 固有の Tensile カーネルをビルドし、`ROCBLAS_TENSILE_LIBPATH` を設定してください。

**ログに `FlashAttention:Auto` と表示される**

これは期待通りの動作です。`OLLAMA_FLASH_ATTENTION` 環境変数の値に関わらず、gfx900 では Flash Attention はコード上で無効化されています。

---

## ソースリポジトリ

パッチ済み Ollama ソースはこちらで管理されています:
[ollama-src / vega-int8-probe ブランチ](../../)

適用済みパッチ:

| 変更内容 | ファイル |
|----------|----------|
| `IsROCmGFX900()` 検出関数 | `ml/device.go` |
| gfx900 での Flash Attention 無効化 | `ml/device.go` |
| gfx900 での `num_parallel=1` 強制 | `server/sched.go` |
| gfx900 での `num_ctx` を 8192 に上限設定 | `server/routes.go` |
| パス検索で `build-gfx900/lib/ollama` を優先 | `ml/path.go` |
| CMake ROCm プリセットに gfx900 を追加 | `CMakePresets.json` |
| Dockerfile の rocBLAS クリーンアップを gfx906 のみに限定 | `Dockerfile` |

---

## 免責事項

これは Vega / gfx900 / MI25 ハードウェア上での調査・検証を目的とした実験的なフォークです。
AMD や Ollama プロジェクトによる公式の互換性声明、製品コミットメント、サポート保証ではありません。

エンドツーエンドの動作確認は、対象のモデルとワークロードを対象ハードウェア上で直接検証した場合にのみ主張してください。
