# GH200 96GB：agentionai版llama.cppとGyro-M／Lのセットアップ

更新日：2026-10-23（日本時間）

## 目的と検証状況

GH200（Grace ARM64 CPU＋Hopper GPU、HBM 96GB）を推論サーバーとして、Qwen3.8-Flash-Next Gyro-M／LをOpenAI互換APIで利用する。用途はコーディング、デバッグ、評価、分析。最初は32K・1リクエストで確認し、64K×2、128K×2へ段階的に拡張する。

**これは配布元の案内に基づく導入・検証手順であり、GH200実機でのビルド成功、速度、長文2並列動作を記録した実績資料ではない。** コマンドのシェル構文は確認したが、CUDAビルドと推論は未実行。実行後は末尾の記録表を埋める。

既存の[GH200＋Cline Remote SSHメモ](gh200-cline-remote-ssh-setup.md)とは別の推論エンジン構成。既存の27B/vLLMを停止・削除する操作は含まないが、同じGPU上で併用するとメモリを競合するため、今回の確認にはGPUを空けておく。

## 構成を選ぶ

| 項目 | Gyro-M | Gyro-L |
|---|---|---|
| ファイル名 | `Qwen3.8-Flash-Next-Gyro-M-TQ2_0.gguf` | `Qwen3.8-Flash-Next-Gyro-L-TQ3_0.gguf` |
| ダウンロード容量 | 約92GB | 約108GB |
| GPU上の重みの公表目安 | 約35GiB | 約50.2GiB |
| 選択方針 | メモリ余裕を優先 | 品質を優先して比較する候補 |

容量は配布元の公表値。N-gram埋め込みをGPU外に置く前提であり、KV cache、再帰状態、一時バッファ等は別途必要。2人でモデルの重みを共有するが、各リクエストの状態は追加される。公表GPUメモリだけから最大コンテキストや速度を保証しない。

**通常版ggml-org/llama.cppではGyroの独自形式を読めないため、配布元が指定するagentionai版を使用する。** Linux ARM上のStrataを前提にしない。GH200でのagentionai版の動作も、本手順で確認する対象となる。

## 1. GH200側：環境確認

Ubuntu／Debian系、Bashを想定する。以下はGH200へSSHログインして実行する。

```bash
uname -m
nvidia-smi
nvcc --version
cmake --version
python3 --version
free -h
df -h "$HOME"
```

- `uname -m`が`aarch64`、GPUがGH200であることを確認。
- `nvidia-smi`のCUDA表示だけではToolkit導入済みと判断しない。`nvcc`が必要。
- `nvcc`がなければ、管理者が提供するCUDA環境やEnvironment Modulesを確認する。ドライバを一括置換する手順はここでは行わない。
- 保存先は高速なローカルNVMe SSDを選ぶ。以下の`$HOME/models`がNFS等なら、ローカルNVMe上のパスへ変更する。
- 空きディスクの作業目安は、片方のモデルで150GB程度、両方で250GB程度。ダウンロードキャッシュやビルド等によって増える。
- CPU RAMの空きも記録する。ディスク読み出し時のページキャッシュ等で使用量が変わる。

## 2. GH200側：agentionai版を取得・ビルド

既存の通常版llama.cppとは別のディレクトリへ取得する。次のパッケージ導入にはsudo権限が必要。

```bash
sudo apt-get update
sudo apt-get install -y \
  build-essential cmake ninja-build git pkg-config \
  libcurl4-openssl-dev libssl-dev python3-venv curl jq

mkdir -p "$HOME/src"

git clone --depth 1 --recurse-submodules --branch main \
  https://gitlab.com/agentionai-group/llama.cpp.git \
  "$HOME/src/llama.cpp-agention"

cd "$HOME/src/llama.cpp-agention"

cmake -S . -B build-gh200 -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DGGML_CUDA=ON \
  -DCMAKE_CUDA_ARCHITECTURES=90

cmake --build build-gh200 -j 16 \
  --target llama-server llama-bench
```

GH200のCompute Capabilityは9.0なので、CUDAアーキテクチャを`90`と指定する。CPU側はGH200上でネイティブビルドする。すでに取得先が存在するときは、上書きせず`git status`と`git remote -v`で内容を確認する。

確認：

```bash
cd "$HOME/src/llama.cpp-agention"
./build-gh200/bin/llama-server --version
./build-gh200/bin/llama-server --list-devices
./build-gh200/bin/llama-server --help > /tmp/gyro-server-help.txt
grep -E 'ngram-on-disk|parallel|kv-unified|cache-type|cont-batching' \
  /tmp/gyro-server-help.txt
git rev-parse HEAD
```

CUDAデバイスと`--ngram-on-disk`を確認できてからモデルをダウンロードする。表示されない場合は、別バイナリ、古いfork、ビルド設定を点検する。未知のオプションを削除して先へ進めず、取得元とリビジョンを確認する。

`main`は更新される。ビルドできたコミットID、CUDA、コンパイラを記録し、比較評価では同じ版を使う。

## 3. GH200側：モデルを取得

ダウンロード用のPython仮想環境を作る。

```bash
python3 -m venv "$HOME/venvs/hf-gyro"
"$HOME/venvs/hf-gyro/bin/pip" install -U huggingface_hub
mkdir -p "$HOME/models/qwen38-gyro"
```

次の**どちらか一方**を選び、同じターミナルでダウンロードする。

Gyro-L：

```bash
GYRO_MODEL_FILE='Qwen3.8-Flash-Next-Gyro-L-TQ3_0.gguf'
```

Gyro-M：

```bash
GYRO_MODEL_FILE='Qwen3.8-Flash-Next-Gyro-M-TQ2_0.gguf'
```

選択後：

```bash
"$HOME/venvs/hf-gyro/bin/hf" download \
  agentionai/Qwen3.8-Flash-Next-Gyro-GGUF \
  "$GYRO_MODEL_FILE" LICENSE README.md \
  --local-dir "$HOME/models/qwen38-gyro"

ls -lh "$HOME/models/qwen38-gyro/$GYRO_MODEL_FILE"
sha256sum "$HOME/models/qwen38-gyro/$GYRO_MODEL_FILE"
```

今回はテキスト用途のため、画像用mmprojとMTP用draftは取得しない。再現性が必要な評価では、モデルのHugging Faceリビジョンも記録し、以後の`hf download`に`--revision <記録したリビジョン>`を追加する。上のSHA-256は取得ファイルの同一性確認に使えるが、配布元のチェックサムとの照合を行ったことにはならない。

## 4. GH200側：32K・1リクエストで起動

以下のシェル関数を、そのままBashに貼り付ける。再ログインしたら、モデル選択とこの関数定義をもう一度実行する。関数はこのメモを読むだけでは自動保存されない。

```bash
run_gyro() {
  : "${GYRO_MODEL_FILE:?先にGyro-MまたはLのファイル名を設定してください}"
  "$HOME/src/llama.cpp-agention/build-gh200/bin/llama-server" \
    --model "$HOME/models/qwen38-gyro/$GYRO_MODEL_FILE" \
    --alias qwen38-gyro \
    --n-gpu-layers 99 \
    --ngram-on-disk \
    --fit off \
    --ctx-size "${GYRO_CTX:-32768}" \
    --parallel "${GYRO_PARALLEL:-1}" \
    --no-kv-unified \
    --cont-batching \
    --cache-type-k "${GYRO_KV:-f16}" \
    --cache-type-v "${GYRO_KV:-f16}" \
    --flash-attn on \
    --batch-size 512 \
    --ubatch-size "${GYRO_UBATCH:-256}" \
    --threads 16 \
    --jinja \
    --temp 1.0 \
    --top-p 0.95 \
    --top-k 20 \
    --min-p 0.05 \
    --host 127.0.0.1 \
    --port 8080
}

GYRO_CTX=32768
GYRO_PARALLEL=1
GYRO_KV=f16
GYRO_UBATCH=256
run_gyro
```

`--ngram-on-disk`でN-gram埋め込みをディスクから読む方式を使う。この構成に、通常GGUF向けの`--lazy-mode off`や`per_layer_token_embd=CPU`を混ぜない。まず配布元の方式で確認する。

MTPは初回には使わない。KV cacheはf16から開始し、量子化の影響と基本動作を切り分ける。`--fit off`で自動調整を止めるため、指定容量が収まらなければ起動エラーになる。エラー時は後述の調整を行う。

## 5. GH200側：APIとGPU使用量を確認

別のSSHターミナルから実行する。

```bash
curl -sS http://127.0.0.1:8080/health
curl -sS http://127.0.0.1:8080/v1/models | jq .

curl -sS http://127.0.0.1:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "qwen38-gyro",
    "messages": [
      {
        "role": "user",
        "content": "次のCコードのバグを説明し、最小修正と境界条件のテストを示してください。\nint sum(const int *a, int n) { int s=0; for(int i=0; i<=n; ++i) s+=a[i]; return s; }"
      }
    ],
    "max_tokens": 8192,
    "temperature": 1.0,
    "top_p": 0.95
  }' | jq .
```

正常な日本語とコードが返り、`i <= n`の範囲外アクセスを指摘できるか確認する。この1問だけでモデル品質を判定せず、実際の修正結果をビルド・テストする。

```bash
watch -n 1 nvidia-smi
```

forkにダッシュボードがある版では`http://127.0.0.1:8080/dashboard`からスロットや速度を確認できる。まず起動ログと`nvidia-smi`を確認し、ダッシュボードの有無に依存しない。

## 6. GH200側：2人共有と長いコンテキスト

前景起動中のサーバーを`Ctrl+C`で止める。モデルは1つだけロードし、2件のリクエストを同時処理する。

| 段階 | GYRO_CTX（合計） | GYRO_PARALLEL | 1件あたりの上限目安 |
|---|---:|---:|---:|
| 初回 | 32768 | 1 | 32K |
| 2並列の確認 | 131072 | 2 | 64K |
| 目標・未検証 | 262144 | 2 | 128K |

最初に64K×2：

```bash
GYRO_CTX=131072
GYRO_PARALLEL=2
run_gyro
```

安定後、停止して128K×2：

```bash
GYRO_CTX=262144
GYRO_PARALLEL=2
run_gyro
```

本手順は`--no-kv-unified`で割り当てを明示している。起動ログの`n_slots`と`n_ctx_slot`を確認し、128K×2なら各スロットが131072になっていることを確認する。コンテキストにはコード、会話、ツール出力、推論・回答用のトークンが含まれるため、入力で上限を使い切らない。

メモリ不足なら停止してKV cacheをq8_0へ変更し、再起動する。

```bash
GYRO_KV=q8_0
run_gyro
```

入力処理中の一時メモリ不足には、ubatchを下げて再試行する。

```bash
GYRO_UBATCH=128
run_gyro
```

2人のエージェントがそれぞれ複数リクエストを並列発行すると、2スロットでは待ちが生じる。最初は各クライアントのLLM同時リクエスト数を1にする。スロットはユーザー専用アカウントではなく、会話履歴は各クライアントが管理する。

両方の会話を上限近くまで伸ばし、同時に実行して確認する。片方の長文入力中にもう片方の生成が止まらないか、TTFT、1人あたりの生成速度、メモリの最大値、タスク完了率を記録する。「サーバー起動成功」だけで長文2並列の検証完了としない。

## 7. 各利用者のPC：SSHトンネルとエージェント設定

各PCで実行する。`USER`と`GH200_HOST`は自分の接続先に置き換える。

```bash
ssh -N -L 18080:127.0.0.1:8080 USER@GH200_HOST
```

| 項目 | 設定 |
|---|---|
| Provider | OpenAI Compatible |
| Base URL | `http://127.0.0.1:18080/v1` |
| Model ID | `qwen38-gyro` |
| API key入力必須のクライアント | `local`等の任意文字列。今回のサーバー設定では認証なし |
| Context length | 実測確認済みの1件あたりの値（目標は131072） |

2人とも自分のPCで同じローカルポート18080を使える。SSH接続は別々に作る。エージェント自体が別の開発サーバー上で動く場合は、そのリクエスト送信元からGH200に届くトンネル／経路を設定する。PCのlocalhostは開発サーバーのlocalhostではない。

サーバーは127.0.0.1で待ち受ける。推論を行うGH200と、ソースコードを編集・ビルドする開発サーバーは別にできる。

## 8. MとLを切り替える・再開する

起動中のサーバーを止め、3節で別モデルを選んでダウンロードする。`GYRO_MODEL_FILE`を変更後、`run_gyro`で再起動する。同じエイリアスを使うのでクライアントのModel IDは変更不要だが、評価記録にはM/Lを必ず書く。

SSH切断後も動かしたい場合は、既存の[SSH＋tmuxメモ](linux/ssh-tmux-session.md)を参照する。tmuxの新しいシェルではモデル変数と関数の再定義が必要。まず手動で安定動作を確認し、その後に常駐化を検討する。

## 9. 問題の切り分け

| 症状 | 確認・対応 |
|---|---|
| `nvcc`がない | CUDA Toolkitの場所／moduleを確認。ドライバ認識とは分けて確認 |
| CUDAコンパイラが見つからない | インストール先の`nvcc`を確認し、CMakeに`-DCMAKE_CUDA_COMPILER=/実際のパス/bin/nvcc`を渡す |
| CMake最小バージョン／GCC互換性エラー | 取得版のエラーとビルド文書に従い、Toolkitに対応するツールチェーンを選ぶ |
| CUDAビルドがメモリ不足 | `cmake --build build-gh200 -j 4 --target llama-server llama-bench`で並列数を下げる |
| ARM上でx86 intrinsics等のコンパイルエラー | 最初のエラー、コミット、コンパイラを保存。ARM移植の追加修正が必要な可能性があり、成功扱いしない |
| 未知のGGUF型／`ngram-on-disk`がない | forkの取得元・リビジョン・起動バイナリを確認。通常版llama.cppを起動していないか確認 |
| ロード時にGPUメモリ不足 | 他プロセス、選択モデル、N-gram読み出し設定を確認。32K×1へ戻す |
| 長文時にGPUメモリ不足 | KVをq8_0、ubatchを128、コンテキスト縮小を順に比較 |
| 同時実行だけ失敗 | 最新の配布元修正を確認し、まず1並列へ戻して切り分ける |
| 出力の崩れ・無限反復 | 短い入力、f16 KV、1並列、MTPなしの基準構成と比較。サンプリング変更だけで隠さない |
| APIへ接続できない | GH200上のhealth、ポート競合、SSHトンネル、クライアント送信元を順に確認 |

Gyro-M/LのCUDAカーネルがHopperで正しく実行されるかは実機確認事項。別GPUの速度をGH200の予測値として扱わない。

## 10. 実行後の記録

| 項目 | 結果 |
|---|---|
| 実行日／OS | 未記録 |
| GPU／空きHBM／CPU RAM | 未記録 |
| Driver／CUDA Toolkit／GCC／CMake | 未記録 |
| agentionai版コミットID | 未記録 |
| モデル名／HFリビジョン／SHA-256 | 未記録 |
| ビルド・ロード・短文応答 | 未検証 |
| KV型／総コンテキスト／スロット数 | 未記録 |
| 64K×2／128K×2の同時実行 | 未検証 |
| TTFT／入力速度／1人あたり生成速度 | 未測定 |
| 最大HBM／RAM使用量 | 未測定 |
| 27Bと同じ課題でのテスト成功率・所要時間 | 未測定 |

## 参照

- [Gyro配布元：モデルと実行案内](https://huggingface.co/agentionai/Qwen3.8-Flash-Next-Gyro-GGUF)
- [モデルファイル一覧](https://huggingface.co/agentionai/Qwen3.8-Flash-Next-Gyro-GGUF/tree/main)
- [agentionai版llama.cpp：配布元が指定するGitLab](https://gitlab.com/agentionai-group/llama.cpp)
- [agentionai版ビルド文書：GitHub側](https://github.com/agentionai/llama.cpp/blob/main/docs/build.md)
- [agentionai版サーバー文書：GitHub側](https://github.com/agentionai/llama.cpp/blob/main/tools/server/README.md)
- [NVIDIA：GH200のCompute Capability](https://developer.nvidia.com/cuda/gpus)
- [Qwen Community 1.0原文](https://huggingface.co/Qwen/Qwen3.8-Flash-Next/blob/main/LICENSE)

GitHub側文書とGitLabのmainは更新タイミングが異なる可能性があるため、実際に取得した版の`--help`とエラーを優先する。モデルはQwen Community 1.0を継承し、推論エンジンのライセンスとは別。業務利用条件はモデルのLICENSE原文で確認する。
