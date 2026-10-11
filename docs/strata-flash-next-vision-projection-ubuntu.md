# Strata＋Flash-Next UD-IQ4_XSでVisionとexperimental speed projectionを併用する

Ubuntuで動作済みのQwen3.8-Flash-Next（Unsloth UD-IQ4_XS）にprojectionを適用し、その後GPU Visionを追加した。2026年10月12日（日本時間）、起動、画像への応答、Monitorの `Done` ＋ `ESP` 表示まで確認完了の報告があった。

## 1. 前提と到達点

| 項目 | 今回の構成・確認 |
| --- | --- |
| OS / GPU | Ubuntu 26.04.1 LTS / RTX PRO 6000 Blackwell Workstation Edition、96GB |
| RAM | 購入構成128GB |
| Strata | 初期セットアップ時のコミット `fb58e0d`。projection確認画面のエンジンはv0.1.41 |
| モデル | Qwen3.8-Flash-Next、Unsloth UD-IQ4_XS |
| コンテキスト / KV cache | 32,768 tokens / int8 |
| Vision | GPUエンコーダー、最大1,024 image tokensを指定 |
| Projection | layers 4–44、scale 1.0、project / per-layer |
| 最終起動設定 | `~/AI/Strata/experiment-speed-projection-vision.json` |
| UI | UbuntuのFirefoxで `http://127.0.0.1:8080` |

初期導入は[Strata＋Flash-NextのUbuntuセットアップ](strata-flash-next-ubuntu-setup.md)を参照。この手順はその動作済みインストールから進める。IPアドレス・ユーザー名・ホスト名・実機設定JSONの全文は掲載しない。

Vision＋projectionの最終確認は利用者からの完了報告に基づく。画像回答の全文、画像処理時の速度・VRAM・RAM実測値、長時間の安定性は今回採取していない。

## 2. ProjectionとVisionの役割

experimental speed projection（ESP）は、推論中の残差表現から特定の方向を取り除くcontrol vector。配布元はrefusal-direction projectionとして説明しており、拒否傾向と通常の回答の両方が変わり得る。モデルのGGUF重みを書き換える処理ではない。名前にspeedとあるが、高速化や精度改善を保証する機能ではない。

Visionは画像をモデルへ渡すための画像エンコーダー。ESPは言語モデル側の処理、Visionは画像入力の経路なので、設定を組み合わせて動作確認した。画像を生成する機能ではなく、画像に対して文章で回答する機能。

公式資料の目安では、BF16画像エンコーダーは約0.9GBのダウンロードで、GPU利用時に約1.4GBのVRAM枠を使う。これは公式の目安であり、今回のESP＋Vision構成を測定した値ではない。

### 今回、手動でESP設定を追加した理由

`fb58e0d` の `setup.py` はESPの自動設定を `qwen` / `coder` familyに限定している。`unsloth` に `--experimental-speed-projection on` を渡すと未検証としてOFFにするため、UD-IQ4_XSでは別のJSONにエンジン引数を追加した。

同コミットのUD-IQ4_XSには `vision: True` が設定されており、GPU Visionのセットアップは利用できる。セットアップを再実行するたびにESPも自動でONになるとは考えず、起動に使うJSONとMonitorを確認する。

## 3. テキストでprojection ONを確認する

### 3.1 通常設定を保存

Ubuntu側の端末で実行する。

```bash
cd ~/AI/Strata
strata_backup_dir="$HOME/AI/Strata-backup-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$strata_backup_dir"
cp -a run-unsloth-ud-iq4_xs.sh strata-unsloth-ud-iq4_xs.json "$strata_backup_dir/"
git rev-parse HEAD > "$strata_backup_dir/commit.txt"
```

### 3.2 ESP用JSONを作成

すでに `experiment-speed-projection.json` があり、ESPで推論できている場合は、この作成処理は省略して第4節へ進む。以下は新規作成用で、同名ファイルがあれば上書きせず停止する。

```bash
cd ~/AI/Strata
python3 - <<'PY'
import json
from pathlib import Path

root = Path.cwd()
src = root / "strata-unsloth-ud-iq4_xs.json"
dst = root / "experiment-speed-projection.json"
vector = root / "data/experimental-speed-projection/Qwen3.8-Flash-Next-experimental-speed-projection.gguf"

assert vector.is_file(), f"投影ファイルがありません: {vector}"
cfg = json.loads(src.read_text(encoding="utf-8-sig"))
assert "--control-vector-scaled" not in cfg["args"], "元設定に投影が登録済みです"
cfg["args"] += [
    "--control-vector-scaled", f"{vector}:1.0",
    "--control-vector-layer-range", "4", "44",
    "--cvec-mode", "project",
    "--cvec-dir", "per-layer",
]
cfg["log"] = str(root / "strata-esp-test.log")
with dst.open("x", encoding="utf-8") as f:
    json.dump(cfg, f, ensure_ascii=False, indent=2)
print(f"作成しました: {dst}")
PY
```

### 3.3 既存のStrataを停止して起動

既存サーバーがtmuxの `strata` セッションで動作中なら、attachしてStrataのログ画面でCtrl+Cを押す。

```bash
tmux attach -t strata
```

停止後に起動する。直接起動するtmuxセッションなので、Strataの終了時にはセッションも終了する。

```bash
tmux new-session -s strata -c "$HOME/AI/Strata" \
 './.venv/bin/python serve/server.py --engine strata --config experiment-speed-projection.json --port 8080 --open'
```

Aboutでlayers 4–44のprojectionが読み込まれていることを確認し、Chatから質問を送る。MonitorのRecent requestsに `Done` ＋ `ESP` が表示されれば、そのリクエストにESPが適用されている。

今回のVision追加前の画面では、出力154 tokens、生成速度156.0 tok/s、所要時間1.2秒を確認した。この1件からOFFより高速とは判断できず、Vision追加後の速度でもない。

## 4. GPU Visionを追加する

### 4.1 ESP設定もバックアップ

```bash
cd ~/AI/Strata
strata_backup_dir="$HOME/AI/Strata-backup-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$strata_backup_dir"
cp -a \
  run-unsloth-ud-iq4_xs.sh \
  strata-unsloth-ud-iq4_xs.json \
  experiment-speed-projection.json \
  "$strata_backup_dir/"
```

動作中のStrataを停止する。

```bash
tmux attach -t strata
```

Strataの起動ログ画面でCtrl+Cを押す。別の端末で起動している場合は、その起動端末で停止する。

### 4.2 Visionの準備

```bash
cd ~/AI/Strata
export PATH="/usr/local/cuda-13.1/bin:$PATH"
export LD_LIBRARY_PATH="/usr/local/cuda-13.1/lib64${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"

bash ./setup.sh --setup \
  --family unsloth \
  --model UD-IQ4_XS \
  --context 32768 \
  --kv int8 \
  --vision gpu \
  --vision-tokens 1024 \
  --port 8080 \
  --no-start \
  --yes
```

CUDAのパスは今回の13.1配置に合わせたもの。既存のモデルファイルは通常再利用され、画像エンコーダーを追加する。ソースビルド構成では、初回の画像エンコーダービルドに時間がかかることがある。

`--no-start` は準備までで停止する指定。`All set` と通常設定・起動スクリプトの生成を確認してから次へ進む。通常の `strata-unsloth-ud-iq4_xs.json` はVision付きで再生成されるが、そこにESPが自動追加されるわけではない。

### 4.3 ESP＋Vision用の別JSONを作る

**この処理が終わるまで、次の起動コマンドへ進まない。** 既存ESP設定の引数を保持し、セットアップが生成したVision設定を移す。エンジン側の `--vision` も追加する。

```bash
cd ~/AI/Strata
python3 - <<'PY'
import json
from pathlib import Path

root = Path.cwd()
def read(name):
    return json.loads((root / name).read_text(encoding="utf-8-sig"))

cfg = read("experiment-speed-projection.json")
prepared = read("strata-unsloth-ud-iq4_xs.json")

assert "--control-vector-scaled" in cfg["args"], "projection設定がありません"
assert prepared.get("vision", {}).get("gpu") is True, "GPU Vision設定がありません"

cfg["vision"] = prepared["vision"]
for key in ("exe", "mmproj", "model"):
    assert Path(cfg["vision"][key]).is_file(), f"ファイルがありません: {key}"

if "--vision" not in cfg["args"]:
    cfg["args"].append("--vision")
if "--vram-reserve-mib" not in cfg["args"]:
    cfg["args"] += ["--vram-reserve-mib", "700"]

cfg["lib_dirs"] = list(dict.fromkeys(
    cfg.get("lib_dirs", []) + prepared.get("lib_dirs", [])
))
cfg["log"] = str(root / "strata-esp-vision.log")

dst = root / "experiment-speed-projection-vision.json"
with dst.open("x", encoding="utf-8") as f:
    json.dump(cfg, f, ensure_ascii=False, indent=2)

print(f"作成しました: {dst}")
PY
```

`作成しました: .../experiment-speed-projection-vision.json` の表示を確認する。`open("x")` は同名ファイルを上書きしないため、再実行時にFileExistsErrorになったら既存設定を確認する。

### 4.4 起動

```bash
tmux new-session -s strata -c "$HOME/AI/Strata" \
 './.venv/bin/python serve/server.py --engine strata --config experiment-speed-projection-vision.json --port 8080 --open'
```

起動に使うファイルの違い：

| ファイル | 用途 |
| --- | --- |
| `strata-unsloth-ud-iq4_xs.json` | セットアップが生成する通常設定。この手順の実施後はVision付き |
| `experiment-speed-projection.json` | Vision追加前に動作確認したESP設定 |
| `experiment-speed-projection-vision.json` | 今回のESP＋GPU Vision設定 |
| `run-unsloth-ud-iq4_xs.sh` | 通常JSONを参照する生成済み起動スクリプト。ESP＋Vision用JSONを指定する上記コマンドとは異なる |

別JSONは `strata-*.json` という名前にせず、セットアップが別のインストール済みモデル設定として拾わない名前にしている。

## 5. 画像応答とESPの適用を確認

Ubuntu上のFirefoxで `http://127.0.0.1:8080` を開く。

1. AboutでImagesが有効、projectionがlayers 4–44と表示されることを確認する。
2. Chatに写真を添付し、`この画像に写っているものを日本語で説明してください。` と送信する。回答が添付画像の内容と合っているか確認する。
3. Monitorでその画像リクエストが `Done` ＋ `ESP` になっていることを確認する。

2026年10月12日06:07（日本時間）、上記3まで完了したとの報告があった。**UD-IQ4_XS＋GPU Vision＋projection ONでの画像応答を確認済み。**

必要に応じて同じ画像・質問でSamplingのESPスイッチをON/OFFにし、内容と生成速度を比較する。設定を変えると会話キャッシュが一度破棄されるため、速度比較では再利用条件にも注意する。今回、ON/OFFの比較測定は実施していない。

## 6. 実際に出たエラーと対処

### ESP＋Vision用JSONがなく、起動直後に終了

```text
FileNotFoundError: [Errno 2] No such file or directory: 'experiment-speed-projection-vision.json'
```

Visionのセットアップは `All set` まで完了していたが、起動に指定したJSONが見つからなかった。第4.3節の作成処理が未実行か、作成に失敗した可能性があった。

**Visionセットアップをやり直さず、第4.3節のJSON作成処理を実行してから起動し直すことで起動成功の報告があった。**

### tmuxの `[exited]` と `no sessions`

起動コマンドが上のエラーで終了したため、直接そのコマンドを実行していたtmuxセッションも終了した。`tmux attach -t strata` の `no sessions` はこの結果であり、tmuxそのものの障害ではなかった。

起動エラーがすぐ消えて読めない場合は、Ubuntuの端末で直接起動してエラーを確認する。

```bash
cd ~/AI/Strata
./.venv/bin/python serve/server.py --engine strata \
  --config experiment-speed-projection-vision.json --port 8080 --open
```

直接起動したサーバーをCtrl+Cで停止してから、tmuxで起動し直す。

## 7. 次回起動・停止・以前のESP設定へ戻す

次回も第4.4節の起動コマンドを使う。モデルと画像エンコーダーを毎回セットアップする必要はない。すでに `strata` セッションがある場合は `tmux attach -t strata` で状態を確認し、起動済みのサーバーを重複起動しない。

- 動かしたまま端末から離れる：Ctrl+bを押して離し、その後d。
- 戻る：`tmux attach -t strata`。
- 停止：Strataのログ画面でCtrl+C。

Vision追加前のESP構成へ戻す場合は、現在のサーバーを停止し、第3.3節の `experiment-speed-projection.json` を指定したコマンドで起動する。

300W運用は別途 `nvidia-smi` で確認する。[300W上限を起動時に自動適用する設定](linux/ubuntu-nvidia-power-limit-systemd.md)も参照。

## 関連手順・参照した実装

- [Strata＋Flash-Nextの初期セットアップと再起動](strata-flash-next-ubuntu-setup.md)
- [SSHとtmux](linux/ssh-tmux-session.md)
- [UbuntuのRDPと日本語入力](linux/ubuntu-rdp-japanese-input.md)
- [300W制限の自動適用](linux/ubuntu-nvidia-power-limit-systemd.md)
- [Strata setup.py（確認したコミット）](https://github.com/Niko1221/Strata/blob/fb58e0d/setup.py)
- [Strata server.py（同コミット）](https://github.com/Niko1221/Strata/blob/fb58e0d/serve/server.py)
- [VisionとESPの公式説明（同コミット）](https://github.com/Niko1221/Strata/blob/fb58e0d/docs/DETAILS.md)
- [ESPベクトルの説明（同コミット）](https://github.com/Niko1221/Strata/blob/fb58e0d/data/experimental-speed-projection/README.md)
