# RTX PRO 6000でGLM-5.3-Flash Uncensored GGUFを動かす：llama.cppからMayaへの移行と測定

検証日：2026-10-10（日本時間）。

OrcaRouterのGLM-5.3-Flash-Uncensored-GGUF Q3_K_Mを、まずllama-serverでCPU＋GPUに分散して動かした。その後、同じGGUFをProject Mayaへ指定して起動し、異なる内容の質問でも文章生成と速度を確認した。

**結果：Maya v1.0.30で文章生成に成功。測定した応答の生成速度は35.44～54.2 tokens/sだった。** Monitorではexpertのほぼ全量がVRAM＋RAMに配置され、今回の応答中のSSDからのexpert読み出しは表示上0.00個/tokenだった。

これは特定の短い質問による動作確認。厳密な性能比較、回答品質の定量評価、Uncensored化による拒否挙動の評価は実施していない。

通常版Maya＋Visionの導入は[既存のセットアップ記録](project-maya-ubuntu-vision-setup.md)を参照。

## 1. 検証環境と対象モデル

| 項目 | 内容 |
| --- | --- |
| OS | Ubuntu（これまでのセットアップ記録では26.04 LTS） |
| GPU | NVIDIA RTX PRO 6000 Blackwell、VRAM公称96GB |
| CPU | Intel Core Ultra 7 270K Plus |
| RAM | 128GB（32GB×4） |
| モデルの保存先 | ローカルNVMe SSD |
| モデル | orcarouter/GLM-5.3-Flash-Uncensored-GGUF |
| 量子化 | Q3_K_M、約153GB、4分割 |
| llama.cpp | PR #27754のheadからビルドした対応版。実機のcommit SHAは未記録 |
| Project Maya | v1.0.30（Web画面で確認） |
| テスト中のcontext | 8192 tokens |
| 接続 | 接続先Ubuntu内のブラウザ → http://127.0.0.1:8081 |

llama-serverとMayaの測定で使用したGGUFは同じ。Maya用にモデルを再ダウンロード・再量子化する必要はなかった。

実機で確認した配置：

- Maya：`~/AI/project-maya`
- llama.cpp：`~/llm-test/llama-glm-uncensored`
- GGUF：`~/models/glm53-flash-uncensored/Q3_K_M/`
- 先頭ファイル：`GLM-5.3-Flash-Uncensored-Q3_K_M-00001-of-00004.gguf`

ユーザー名・ホスト名を含む絶対パスや認証トークンは掲載していない。

## 2. 最初に確認すること

モデルの保存先に180GB以上の空きを見込む。GPU・RAMを空けるため、Maya、Strata、ComfyUIなど、他の重い処理は停止してから試す。

```bash
nvidia-smi
free -h
df -h "$HOME"
nvcc --version
```

CUDA Toolkitの`nvcc`はllama.cppのGPUコードをビルドするためのコンパイラ。モデル取得用Python venvとは別に扱う。

`nvcc`が見つからない場合は、追加インストールをする前に既存のToolkitを確認する。

```bash
ls -l /usr/local/cuda*/bin/nvcc
```

例えば`/usr/local/cuda/bin/nvcc`が存在するなら、PATHへ追加する。

```bash
export PATH="/usr/local/cuda/bin:$PATH"
nvcc --version
```

実際の配置が`/usr/local/cuda-13.0/bin/nvcc`などなら、そのパスに合わせる。このPATH変更は同じ端末内で有効。

## 3. llama-serverのビルド

以下は今回使用した対応版を新規取得する手順。PRのheadは更新され得るため、再実行時に同じcommitになるとは限らない。これは検証時の手順の記録であり、将来の最新推奨版を意味しない。

```bash
sudo apt update
sudo apt install -y \
  git cmake build-essential libcurl4-openssl-dev \
  python3-venv

mkdir -p ~/llm-test
cd ~/llm-test

git clone https://github.com/ggml-org/llama.cpp.git llama-glm-uncensored
cd llama-glm-uncensored

git fetch origin pull/27754/head
git checkout -b glm53-compatible FETCH_HEAD

cmake -S . -B build \
  -DGGML_CUDA=ON \
  -DCMAKE_BUILD_TYPE=Release

cmake --build build -j 8 --target llama-server
```

### CMakeがnvccを見つけられない場合

今回、Toolkitが存在していても、CMakeのCUDAコンパイラ検出で停止した。続く「Makefileがありません」は、その設定失敗の結果だった。

`nvcc`の配置を確認し、CMakeへ直接指定して解消した。

```bash
/usr/local/cuda/bin/nvcc --version

cd ~/llm-test/llama-glm-uncensored
cmake --fresh -S . -B build \
  -DGGML_CUDA=ON \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_CUDA_COMPILER=/usr/local/cuda/bin/nvcc

cmake --build build -j 8 --target llama-server
```

今回はこの手順でllama-serverのビルドに成功した。Toolkitが別の場所なら`CMAKE_CUDA_COMPILER`も変更する。

## 4. Q3_K_Mのダウンロード

### Python venvと認証

```bash
python3 -m venv ~/llm-test/hf-env
source ~/llm-test/hf-env/bin/activate
pip install -U huggingface_hub
hf auth login
```

[モデルページ](https://huggingface.co/orcarouter/GLM-5.3-Flash-Uncensored-GGUF)でログインし、表示されるアクセス条件を確認して同意する。端末の認証には同じアカウントの読み取り権限を持つトークンを使う。

トークンは端末へ入力し、GitHubや共有ログには保存しない。ここではアクセス条件の法的解釈は行わず、配布ページと申請画面の現行の記載を確認する。

### 対象ファイルを確認して取得

```bash
mkdir -p ~/models/glm53-flash-uncensored
df -h ~/models/glm53-flash-uncensored

hf download orcarouter/GLM-5.3-Flash-Uncensored-GGUF \
  --include "*Q3_K_M*.gguf" \
  --local-dir ~/models/glm53-flash-uncensored \
  --dry-run
```

Q3_K_Mの4分割ファイルが表示されたら、実際に取得する。

```bash
hf download orcarouter/GLM-5.3-Flash-Uncensored-GGUF \
  --include "*Q3_K_M*.gguf" \
  --local-dir ~/models/glm53-flash-uncensored
```

今回、このダウンロードに成功した。画像入力用mmprojはこの取得対象に含めていない。

### 最初のinclude指定で起きたエラー

最初の案内では`--include`の後ろに2つのパターンを置いたが、今回使用したCLIでは2つ目が実ファイル名として解釈されて失敗した。上記のように**1つのパターンへまとめて解消**した。

ログインと保存先の空き容量には問題がなく、venvや認証を作り直す必要はなかった。

## 5. llama-serverでGPU＋RAMへ分散して起動

分割GGUFは先頭ファイルを指定する。残りの分割ファイルも同じ場所にそろえておく。

```bash
cd ~/llm-test/llama-glm-uncensored

GLM_MODEL_FILE="$(find "$HOME/models/glm53-flash-uncensored" \
  -type f -name '*Q3_K_M*00001-of-*.gguf' -print -quit)"

printf 'モデル: %s\n' "$GLM_MODEL_FILE"
```

`モデル:`の後ろに対象パスが表示されたことを確認して、**同じ端末で**次を実行する。表示が空なら起動せず、保存先とファイル名を確認する。

```bash
NVIDIA_TF32_OVERRIDE=0 ./build/bin/llama-server \
  -m "$GLM_MODEL_FILE" \
  --n-gpu-layers 999 \
  --n-cpu-moe 24 \
  -c 8192 \
  -np 1 \
  -t 8 \
  -b 128 \
  -ub 64 \
  -fa off \
  --jinja \
  --host 127.0.0.1 \
  --port 8081
```

| 指定 | 意味 |
| --- | --- |
| `--n-gpu-layers 999` | 原則として層をGPUへ配置 |
| `--n-cpu-moe 24` | 最初の24層のMoE重みをCPU側へ配置 |
| `-c 8192` / `-np 1` | 8K context、同時処理1会話 |
| `-t 8` | CPU計算のスレッド数 |
| `-b 128` / `-ub 64` | 入力処理のbatch／microbatch |
| `NVIDIA_TF32_OVERRIDE=0` / `-fa off` | 今回使ったPR版の案内に合わせた指定 |

会話中の「CPU 20／19／18」は**CPUスレッド数ではなく、`--n-cpu-moe`の層数**を指している。`-t`の変更記録はない。

起動後、Ubuntu内のブラウザで`http://127.0.0.1:8081`を開き、次を送信した。

```text
日本語で、GPUとCPUの違いを3文で説明してください。
```

今回、文章生成に成功した。

## 6. llama-serverの調整結果

`--n-cpu-moe`を減らすとCPU側のexpertがGPUへ移るため、VRAM使用量が増える。変更するたびに起動端末でCtrl+Cを押し、値を変えて再起動した。

| --n-cpu-moe | 表示された生成速度 | VRAM空きの目安 | 記録上の扱い |
| ---: | ---: | ---: | --- |
| 24 | 17.49 tokens/s | 約20.1GiB | 初期設定 |
| 20 | 20.18 tokens/s | 約7.27GiB | 改善を確認 |
| 19 | 20.97 tokens/s | 約4.07GiB | 最後の画像の設定を会話上19と推定。実行コマンドでの再確認はしていない |
| 18 | 21.94 tokens/s | 約0.88GiB | ユーザーが18と明示。VRAMの余裕が小さい |

初期の別の応答では約12 tokens/sも表示された。上表は会話で比較に使った応答の値。

- 24→20では、代表値17.49→20.18 tokens/s、約15%増。
- 20→18では、20.18→21.94 tokens/s、約9%増。
- 18はVRAMの空きが小さいため、会話では19を当面の設定候補とした。
- 各条件の反復数や生成長は統一していないため、厳密な速度差として扱わない。

別端末で使用量を確認するコマンド：

```bash
nvidia-smi --query-gpu=memory.used,memory.total --format=csv
free -h
```

`nvidia-smi`のMiBをGiBへ換算した値と、Maya MonitorのGB表記はそのまま同一視しない。また、RAMの`free`だけで不足と判断せず、`available`やファイルキャッシュも確認する。

## 7. 同じGGUFをMayaへ指定する

Mayaは通常版GLM-5.3-Flashで導入・起動済みだった。今回の移行では同じMayaの環境を使い、取得済みUncensored GGUFを指定した。

Maya公式の量子化モデル以外を`--gguf-dir`で読み込む機能は実験的。今回のQ3_K_Mでは起動と文章生成に成功した。

### 起動

1. llama-serverをCtrl+Cで止める。通常版Mayaなど、他のモデルサーバーも止める。
2. Mayaのディレクトリへ移動し、先頭GGUFを指定する。

```bash
cd ~/AI/project-maya

GLM_MODEL_FILE="$(find "$HOME/models/glm53-flash-uncensored" \
  -type f -name '*Q3_K_M*00001-of-*.gguf' -print -quit)"

printf 'モデル: %s\n' "$GLM_MODEL_FILE"
```

パスが表示されたことを確認し、同じ端末で実行する。

```bash
./maya.sh \
  --setup \
  --gguf-dir "$GLM_MODEL_FILE" \
  --gpu 0 \
  --context 8192 \
  --port 8081 \
  --no-vision \
  --plain
```

- `--gguf-dir`には先頭ファイルを指定した。
- 最初の互換性確認を文章生成に絞り、`--no-vision`を指定した。
- `--plain`は端末ログを読みやすくするため。
- llama-serverの`--n-cpu-moe`などをMayaへ渡す必要はない。
- 初回はGGUFのあるフォルダーへpack（読み込み用インデックス）を作るため、そのフォルダーに書き込み権限が必要。モデル本体の再ダウンロードではない。

最初の案内には`~/project-maya`と記載したが、実機の配置は`~/AI/project-maya`だった。この記録のコマンドは実機の配置に合わせている。

### 作成されたファイルと起動成功

起動ログで確認したファイル：

```text
~/AI/project-maya/maya-uncensored-q3_k_m.json
~/AI/project-maya/run-maya-uncensored-q3_k_m.sh
~/AI/project-maya/maya-uncensored-q3_k_m.log
```

ログでは`ready: http://127.0.0.1:8081/`が表示され、context 8192で回答が返った。最初の応答は154 tokens、35.44 tokens/s、expert cache 86.6% hit。

セットアップ中の自動チューニングは、GPU 0に1744MiBの利用があるためスキップされ、`this PC is NOT tuned`と表示された。その後の起動と文章生成は成功した。1744MiBの利用元は今回特定しておらず、これだけで別のモデルサーバーが残っていたとは断定しない。

### 次回起動の参考

モデル別の生成済み起動スクリプトを使うと、通常版Mayaと今回の設定を選び分けやすい。

```bash
cd ~/AI/project-maya
./run-maya-uncensored-q3_k_m.sh
```

停止は起動端末でCtrl+C。この生成済みスクリプトの存在はログで確認したが、停止後・OS再起動後の実行自体は今回未検証。

## 8. ブラウザに以前のllama-uiが残った問題

Mayaは起動していたが、同じ`http://127.0.0.1:8081`で以前のllama.cpp風UIが表示され、MayaのMonitorが見つからなかった。

画面の見た目だけではサーバーを判定できない。別端末でポートの利用プロセスを確認した。

```bash
ss -ltnp '( sport = :8081 )'
```

今回は`python`が待ち受けていた。これだけではMayaとは確定しないが、Mayaの起動ログの`ready`、生成ログ、設定ファイル名を合わせて、Mayaが応答していることを確認した。

Firefoxの**≡ → 新しいプライベートウィンドウ**から同じURLを開くと、Project MayaのUIとMonitorが表示された。通常ウィンドウに以前のUIのキャッシュ等が残っていた可能性が高いが、保存内容の詳細までは調べていない。

今回のUIでは、**左側のChatの下にMonitor**がある。上部タブを探す案内は実際のUIと一致していなかった。

## 9. Mayaの測定結果

### テストA：GPUとCPUの違い

上記と同じ短い質問を、Mayaの新しいUIで送信した。前の質問と同じ内容なので、expertキャッシュが温まった影響を含む。

### テストB：書店イベントの企画

内容を変え、長めの回答を生成した。

```text
小さな書店で、週末の来店者を増やすイベントを企画してください。予算は3万円、スタッフは2人、準備期間は1週間です。
性質の異なる企画を3つ挙げ、それぞれの対象客、準備内容、予算配分、当日の進め方を説明してください。最後に、実行しやすい案を1つ選び、理由を述べてください。
全体を日本語で800字程度にまとめてください。
```

### Web UI／Monitorの表示値

| 指標 | テストA：GPU・CPU | テストB：書店イベント |
| --- | ---: | ---: |
| 生成tokens | 148 | 695 |
| 応答時間（UIの丸め表示） | 3秒 | 15秒 |
| decode／回答生成 | 54.2 tokens/s | 49.4 tokens/s |
| prefill／入力処理 | 47 tokens/s | 307 tokens/s |
| VRAM hit | 93.8% | 92.3% |
| VRAMのexpert | 7,657個／86.4GB | 7,651個／86.4GB |
| Pinned RAMのexpert | 4,435個／58.0GB | 4,440個／58.0GB |
| SSD onlyのexpert | 4個 | 5個 |
| FROM RAM | 4.26個/token | 5.97個/token |
| FROM SSD | 0.00個/token | 0.00個/token |
| PROMOTED | 4.29個/token | 6.00個/token |

Monitorの`live`表示は12,096 routed experts。表示時点のexpert配置は固定ではなく、応答やバックグラウンドの入れ替えで変わる。

読み方：

- **decode**は回答生成の速度、**prefill**は入力の処理速度。相互に置き換えて比較しない。
- **VRAM hit**はアクセスのヒット率であり、全expertの何%がVRAMに入るかを表す配置比率ではない。
- **VRAM 86.4GB**はexpert配置の表示で、GPU全体の使用量やモデル全体の容量とは異なる。
- **FROM RAM**はRAMからPCIe経由で取り込んだexpert数/token。RAM側expertのすべてがGPUへ転送されるとは限らない。
- **FROM SSD 0.00**は丸められた表示値。SSDを全く使わないという意味ではない。モデルの保存・起動時読み込みにはSSDを使う。
- **SSD only**に数個残っていても、それらが今回の応答で必要にならなければ、生成中のSSD読み出しはほぼ発生しない。

異なる内容のテストBでも49.4 tokens/sを確認できた。この2件では、同じ質問だけを繰り返した場合に限定されず、実用的な生成速度で動いた。

## 10. llama-serverとMayaの違い、速度差の解釈

### 今回の設定での違い

| 項目 | llama-server | Maya |
| --- | --- | --- |
| expertの配置 | 指定した最初のN層のMoE重みをCPU側へ固定し、他をGPUへ配置 | 各層のよく使うexpertをVRAMへ集め、RAM・SSDと階層管理 |
| RAM側expertの計算 | 今回のCPU MoE設定ではCPUで計算 | CPU計算と、GPUへの転送＋計算の配分を調整 |
| 利用状況への適応 | 今回は層数を人手で変更して調整 | expertの利用状況に応じてVRAMへ保持・昇格 |
| 生成中のSSD | mmapのページがRAMにない場合などには読み出しが起き得る | VRAM・RAMにないexpertが必要になったときに読み出す |

これは**今回の設定同士の比較**。llama.cpp全体にexpertキャッシュや投機的デコード等の機能がないという主張ではない。将来版・別オプション・別モデルで結果は変わり得る。

Mayaの35.44 tokens/sは、llama-serverの代表値20.97 tokens/sに対して約1.69倍。キャッシュが温まった同じ短い質問では54.2 tokens/s、比率は約2.58倍。ただし、system prompt、生成内容・長さ、入力履歴、キャッシュ状態などが統一されていないため、これらは画面表示値の比率であり、保証できる性能向上率ではない。

書店イベントの49.4 tokens/sは別の質問なので、llama-serverとの倍率比較には使わない。

### 「先読み」と「投機的デコード」は別

| 仕組み | 先に扱うもの | 今回の解釈 |
| --- | --- | --- |
| expertの先読み・転送の重ね合わせ | 必要になりそうなexpertの重み | Mayaには入力処理中、attention計算と重ねて次の層のexpertをRAMからGPUへ転送するprestageがある |
| 動的expertキャッシュ | よく使うexpertの重み | 今回のVRAM hit約92～94%は、利用されたexpertをVRAMで処理できる割合が高いことを示す |
| 投機的デコード／MTP | 先の出力tokenの候補 | Maya公式説明では2枚以上のGPUで使用。今回のGPU 1枚構成の高速化理由には当たらない |

今回の高速化には、expert配置の変更によってGPUで処理する割合が高まったことや、計算・転送のスケジューリングが寄与した可能性がある。ただし、各要素の有効／無効を比較する測定や詳細なprofilingは実施しておらず、どれが何割寄与したかは確定できない。

**今回の結果を「SSDの投機的読み出しが速いから」と説明するのは適切でない。** 実測した応答では、expertの大部分がVRAM＋RAMにあり、SSD読み出しは表示上ほぼゼロだった。

## 11. 確認した範囲と残る検証

| 項目 | 状態 |
| --- | --- |
| llama-serverのCUDAビルド | 成功 |
| Q3_K_Mの取得・文章生成 | 成功 |
| CPU MoE層数の削減による速度とVRAMの変化 | 確認 |
| 同じUncensored GGUFをMayaへ指定 | 成功 |
| Maya v1.0.30、8K、GPU 1枚での文章生成 | 確認 |
| 内容の異なる質問での生成速度 | 確認：書店イベント49.4 tokens/s |
| Monitorのexpert配置と読み出し | 確認 |
| llama-uiが残る表示問題 | プライベートウィンドウでMaya UIを確認 |
| このUncensoredモデルのVision | 未検証。通常版でのVision成功とは別 |
| MTP／複数GPU | 未検証 |
| 8Kに近い長文context・長時間使用 | 未検証 |
| 回答品質・Uncensored挙動の定量評価 | 未実施 |
| 同条件・反復測定による厳密なエンジン比較 | 未実施 |
| 手動calibrateの完了 | 未実施 |
| 今回のモデル別スクリプトでの再起動・OS再起動後の確認 | 未実施 |

現時点では、この設定で利用を始められるだけの起動・短い文章生成の確認ができた。速度の一般化や、長文・Visionでの動作は追加検証が必要。

## 参照資料

- [OrcaRouter：GLM-5.3-Flash-Uncensored-GGUF](https://huggingface.co/orcarouter/GLM-5.3-Flash-Uncensored-GGUF)
- [llama.cpp：GLM-5-Next対応PR #27754](https://github.com/ggml-org/llama.cpp/pull/27754)
- [llama.cpp：公式ビルド手順](https://github.com/ggml-org/llama.cpp/blob/master/docs/build.md)
- [llama-server：オプションの公式説明](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md)
- [Hugging Face Hub CLI](https://huggingface.co/docs/huggingface_hub/guides/cli)
- [Hugging Face：gated modelの仕組み](https://huggingface.co/docs/hub/models-gated)
- [Project Maya：README・Tuning](https://github.com/mw00/project-maya)
- [Project Maya v1.0.30：README](https://github.com/mw00/project-maya/blob/v1.0.30/README.md)
- [通常版Maya＋Visionのセットアップ記録](project-maya-ubuntu-vision-setup.md)

公式ページやPR headは更新される。再現時は、実機のcommit SHA、`./maya.sh --help`、モデルファイルのrevisionを記録すると比較しやすい。
