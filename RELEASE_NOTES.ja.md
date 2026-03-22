# リリースノート — ollama-gfx900-starter-kit

**ベース:** Ollama v0.18.2
**ブランチ:** `vega-int8-probe`
**ビルドタグ:** `v0.18.2-20-gd44f21d89`
**対象:** AMD Radeon Instinct MI25 / gfx900 (Vega10)
**プラットフォーム:** Linux + ROCm 6.1+

---

## このリリースで追加されるもの

上流の Ollama は gfx900 をランタイムでブロックしています（コミット `55ca82726`）。このキットはそのブロックを取り除き、gfx900 向けにコンパイルされた HIP バックエンドを提供することで、MI25 ハードウェア上での LLM 推論を可能にします。

### 適用済みパッチ（v0.18.2 から 20 コミット）

| 変更内容 | ファイル |
|----------|----------|
| `IsROCmGFX900()` 検出関数 | `ml/device.go` |
| gfx900 での Flash Attention 無効化 | `ml/device.go` |
| gfx900 での `num_parallel=1` 強制 | `server/sched.go` |
| gfx900 での `num_ctx` を 8192 に上限設定 | `server/routes.go` |
| ライブラリパス検索で `build-gfx900/lib/ollama` を優先 | `ml/path.go` |
| CMake ROCm プリセットに gfx900 を追加 | `CMakePresets.json` |
| Dockerfile の rocBLAS クリーンアップを gfx906 のみに限定 | `Dockerfile` |

---

## 収録アーティファクト

`build-gfx900/lib/ollama/` に含まれるファイル:

| ライブラリ | 説明 |
|------------|------|
| `libggml-base.so` / `.so.0` / `.so.0.0.0` | GGML ベース |
| `libggml-hip.so` | gfx900 向けにコンパイルされた HIP/ROCm バックエンド |
| `libggml-cpu-alderlake.so` | CPU バックエンド (Alder Lake) |
| `libggml-cpu-haswell.so` | CPU バックエンド (Haswell) |
| `libggml-cpu-icelake.so` | CPU バックエンド (Ice Lake) |
| `libggml-cpu-sandybridge.so` | CPU バックエンド (Sandy Bridge) |
| `libggml-cpu-skylakex.so` | CPU バックエンド (Skylake-X) |
| `libggml-cpu-sse42.so` | CPU バックエンド (SSE4.2) |
| `libggml-cpu-x64.so` | CPU バックエンド (汎用 x86-64) |

`ollama` バイナリは別途 [Releases](../../releases) ページで配布します。

---

## リポジトリ整理方針（削除ではなく役割分離）

- `ollama-src` はパッチ履歴・実装詳細・検証ドキュメントの正本として維持します。
- `ollama-gfx900-starter-kit` はビルド済み成果物の配布リポジトリとして運用します。
- クリーンアップは、調査履歴を削除する方式ではなく、役割分離とドキュメント整備で実施します。
- 利用者向けに共有する成果物は、ソースリポジトリへのコミットではなく starter-kit のリリース資産で公開します。

---

## 既知の制限事項

**Flash Attention**
Flash Attention は gfx900 で非対応です。`OLLAMA_FLASH_ATTENTION` 環境変数の値に関わらず、コード上で無効化されています。ログに `FlashAttention:Auto` と表示されますが、これは期待通りの動作です。

**並行処理**
`num_parallel` は 1 に強制されます。このハードウェアターゲットでは並行リクエスト処理はサポートされません。

**コンテキスト長**
`num_ctx` の上限は 8192 です。16 GB の MI25 での実用的なデフォルト値は、モデルサイズによって 4096 程度になります。

**rocBLAS / Tensile カーネル**
システムの ROCm パッケージには gfx900 固有の Tensile カーネルが含まれていない場合があり、GEMM 性能に影響することがあります。性能低下や rocBLAS 初期化エラーが発生した場合は、[ROCm-MI25-build](https://github.com/AETS-MAGI/ROCm-MI25-build) でローカルカーネルをビルドしてください。

**Windows**
非対応です。上流の Windows HIP ビルドは gfx900 を除外しており、ここでも回避策は提供しません。

**MLIR iGEMM / XDLops**
これらの計算パスは gfx900 では利用できません。有効なパスは ASM implicit GEMM、DLOPS、non-dot4 フォールバックルートです。

---

## 動作確認

起動後、GPU 検出をログで確認:

```bash
./ollama serve
# 期待されるログ行:
# level=INFO msg="forcing num_parallel=1 for ROCm gfx900" compute=gfx900
```

簡易動作チェック:

```bash
./ollama pull tinyllama
./ollama run tinyllama "Hello"
```

---

## 免責事項

これは Vega / gfx900 / MI25 ハードウェア上での調査・検証を目的とした実験的なリリースです。AMD や Ollama プロジェクトによる公式の互換性声明、製品コミットメント、サポート保証ではありません。

エンドツーエンドの動作確認は、対象のモデルとワークロードを対象ハードウェア上で直接検証した場合にのみ主張してください。
