# RTX PRO 6000：Strata＋QwenからComfyUIをMCPで操作して画像生成する

2026年10月12日（日本時間）、UbuntuのRTX PRO 6000上で、StrataのQwen3.8-Flash-Nextから公式ComfyUI MCPを呼び出し、Z-Image Turboによる画像生成を確認した。MCP導入、実際に使う設定JSONへの登録、32Kのコンテキスト不足への対処、生成完了確認までをまとめる。

## 1. 確認した構成と到達点

| 項目 | 今回の構成・確認 |
| --- | --- |
| OS / GPU | Ubuntu 26.04系 / RTX PRO 6000 Blackwell Workstation Edition、96GB |
| RAM | 購入構成128GB |
| 推論エンジン / LLM | Strata / Qwen3.8-Flash-Next、Unsloth UD-IQ4_XS |
| 既存機能 | GPU Vision、experimental speed projection ON |
| Strata起動設定 | `~/AI/Strata/experiment-speed-projection-vision.json` |
| 最終コンテキスト | 65,536 tokens（32,768から変更） |
| ComfyUI | `~/AI/ComfyUI`、ローカルHTTPサーバー、ポート8188 |
| MCP | Comfy-Orgの`comfy-mcp`と`comfy-cli`、専用venvに導入 |
| MCP結果量 / ラウンド上限 | `max_result_chars: 8000` / `max_rounds: 6` |
| MCPタイムアウト | `timeout_s: 180` |
| 画像生成 | `generate_image`を1回実行し、`prompt_id`の返却を確認 |
| 生成完了 | 状態・出力の確認手順を案内後、利用者から確認完了の報告 |

Strata初期導入は[Ubuntuセットアップ](strata-flash-next-ubuntu-setup.md)、Visionとprojectionは[追加手順](strata-flash-next-vision-projection-ubuntu.md)を参照。この手順は、既に両方のアプリと画像モデルが動作する環境から進める。

ComfyUIは画像生成を実行するサーバー、MCPはその操作をツールとして公開する別プロセス。StrataがMCPクライアントとなり、Qwenがツールを選んで呼び出す。今回のStrataブラウザChatでは、追加のAIエージェントアプリは使っていない。

ユーザー名・ホスト名・実機JSON全文・スクリーンショットは掲載しない。以下のPython例は`Path.home()`から実行者のパスを組み立てる。実機のパッケージバージョン一覧は未採取のため、最新バージョンでの完全再現を保証する記録ではない。

## 2. ComfyUIと画像モデルの前提

ComfyUIのブラウザで、手動の画像生成が成功することを先に確認する。今回、Z-Image TurboとQwen-Image 2.1は、それぞれComfyUIで手動生成を確認済みだった。MCP経由の今回の生成確認に使ったのはZ-Image Turbo。

Z-Image Turbo用の配置は次のとおり。

| ComfyUIからの相対パス | 役割 |
| --- | --- |
| `models/diffusion_models/z_image_turbo_bf16.safetensors` | 画像生成モデル |
| `models/text_encoders/qwen_3_4b.safetensors` | プロンプトのテキストエンコーダー |
| `models/vae/ae.safetensors` | 潜在表現と画像の変換 |

Ubuntuの端末でComfyUIを起動する。既に動いていれば重複起動しない。

```bash
cd ~/AI/ComfyUI
./.venv/bin/python main.py
```

Ubuntu側のブラウザで`http://127.0.0.1:8188`を開く。RDP接続元の別PCで開く場合、`127.0.0.1`は別PC自身を指す。ここでは両アプリを同じUbuntu上で動かす。

## 3. 公式MCPを専用venvにインストールする

ComfyUI本体のvenvとは別に配置する。`uv`は既に導入済みの前提。

```bash
mkdir -p ~/AI/comfy-mcp
uv venv --python 3.12 ~/AI/comfy-mcp/.venv
uv pip install --python ~/AI/comfy-mcp/.venv/bin/python \
  comfy-mcp "comfy-cli>=1.14.0"
```

### トラッキングの確認にはnを選ぶ

最初の対話的なCLI実行では、次の質問が出る場合がある。

```text
Do you agree to enable tracking to improve the application? [y/N]:
```

ローカルMCPの動作にトラッキングへの同意は不要。送信を希望しなければ`n`、または既定のNoを選ぶ。

今回、最初は`y`を選択した。これは利用分析の送信を有効にする設定で、初回だけの送信に限定する選択ではない。公式READMEでは、条件によって検索語も送信対象になると説明している。後から無効にするコマンドは次のとおり。

```bash
COMFY_NO_TELEMETRY=1 ~/AI/comfy-mcp/.venv/bin/comfy tracking disable
```

`Tracking is now disabled.`を確認する。既に送信された情報を削除するコマンドではない。以下のMCP登録でも`COMFY_NO_TELEMETRY=1`を渡す。

既存のComfyUIを既定ワークスペースに指定し、MCPの実行ファイルを確認する。

```bash
COMFY_NO_TELEMETRY=1 ~/AI/comfy-mcp/.venv/bin/comfy set-default ~/AI/ComfyUI
~/AI/comfy-mcp/.venv/bin/comfy-mcp --version
```

今回の構成では`comfy setup`によるクラウド設定は行わない。MCP本体を端末で常時起動しておく必要もない。次の登録後、StrataがstdioのMCPプロセスを起動する。

## 4. 実際に起動する設定JSONへMCPと64K設定を追加する

起動中のStrataを、その起動端末でCtrl+Cにより停止する。ComfyUIは起動したままにする。

**起動コマンドの`--config`で指定するファイルを編集する。** 今回は`experiment-speed-projection-vision.json`。`strata-*.json`という名前だけを頼りに探すと、この手動作成のVision用設定を見落とす。

以下は成功した最終設定をまとめて追加する手順。必要な実行ファイルと既存のコンテキスト指定を確認し、変更前のJSONをバックアップする。既存のVision・projection・モデルパスなどは保持する。

```bash
cd ~/AI/Strata

python3 - <<'PY'
import json
import shutil
from datetime import datetime
from pathlib import Path

path = Path("experiment-speed-projection-vision.json")
cfg = json.loads(path.read_text(encoding="utf-8-sig"))
base = Path.home() / "AI" / "comfy-mcp" / ".venv" / "bin"
workspace = Path.home() / "AI" / "ComfyUI"
for binary in ("comfy-mcp", "comfy"):
    if not (base / binary).is_file():
        raise SystemExit(f"実行ファイルがありません: {base / binary}")
if not (workspace / "main.py").is_file():
    raise SystemExit(f"ComfyUIの配置を確認してください: {workspace}")

args = cfg["args"]
if "--max-context" in args:
    i = args.index("--max-context")
    old_context = args[i + 1]
    args[i + 1] = "65536"
else:
    matches = [i for i, arg in enumerate(args)
               if arg.startswith("--max-context=")]
    if len(matches) != 1:
        raise SystemExit("コンテキスト指定が見つかりません。変更していません。")
    i = matches[0]
    old_context = args[i].split("=", 1)[1]
    args[i] = "--max-context=65536"

cfg.setdefault("mcp_servers", {})["comfyui"] = {
    "command": str(base / "comfy-mcp"),
    "cwd": str(workspace),
    "env": {
        "COMFY_BIN": str(base / "comfy"),
        "COMFY_LOCAL_URL": "http://127.0.0.1:8188",
        "COMFY_NO_TELEMETRY": "1",
    },
}
cfg.setdefault("mcp", {}).update({
    "timeout_s": 180,
    "max_result_chars": 8000,
    "max_rounds": 6,
})

backup = path.with_name(
    path.name + ".before-comfy-mcp-" + datetime.now().strftime("%Y%m%d-%H%M%S")
)
shutil.copy2(path, backup)
path.write_text(json.dumps(cfg, ensure_ascii=False, indent=2) + "\n",
                encoding="utf-8")
print("更新しました:", path)
print(f"コンテキスト: {old_context} → 65536")
print("バックアップ:", backup)
PY
```

`max_result_chars`は1ツール結果の文字数上限で、トークン数ではない。長い結果は切り詰められる。大きなワークフローJSONや一覧を返す操作では、必要な情報まで切れる可能性があるため、保存済みワークフローの指定や絞った検索を使う。

`max_rounds`は1回答中のツール実行ラウンド数の上限。コンテキストの自動要約やVRAM調停を設定する項目ではない。64Kへの変更は会話容量の変更で、モデル重みの再ダウンロードは不要。KV cache等のメモリ使用量は変わり得る。

## 5. Strataを起動し、MCP登録を確認する

```bash
cd ~/AI/Strata
./.venv/bin/python serve/server.py \
  --engine strata \
  --config experiment-speed-projection-vision.json \
  --port 8080 \
  --open
```

起動ログで次を確認する。

- モデルが`ready`まで進み、コンテキストが`65536 tokens`と表示される。
- `MCP server 'comfyui': ... tools (...)`というツール登録の行が出る。ツール数は版によって変わる。
- 失敗した場合はMCP開始エラーの理由を読む。Strata本体の`ready`だけではMCP登録成功とは判断しない。

Ubuntu側のブラウザで`http://127.0.0.1:8080`を開き、再読み込みする。Samplingの`Use tools from MCP servers`をONにする。MonitorにもMCPツールが表示される。

最初は画像生成をせず、次の接続確認だけを送る。

```text
ComfyUIのMCPツールを実際に使って接続を確認してください。
server_infoとsystem_statsを呼び出し、
ComfyUIの接続状態、GPU名、空きVRAMを報告してください。
画像生成はまだ実行しないでください。
取得できなかった情報は推測せず、エラーをそのまま示してください。
```

ツール名、引数、実行結果のブロックを確認する。文章で「接続できた」と回答しただけでは疎通確認にならない。今回、この接続確認の動作成功が報告された。

## 6. 新しいチャットで画像生成を1回だけ実行する

新しいチャットを作る。既存のComfyUIキューに同じ生成が入っていないか確認してから、次を送る。

```text
接続済みのComfyUI MCPのgenerate_imageを1回だけ呼び出してください。

prompt:
A cute tomato character holding a large polished silver roller,
sparkling highlights, colorful cartoon illustration.

wait=falseで実行してください。
モデルやノードの一覧取得、テンプレート探索、
ワークフローの新規作成は不要です。

実行後はprompt_idだけ報告して終了してください。
状態確認の繰り返しや再実行はしないでください。
```

`generate_image`は、公式MCPの標準設定ではZ-Image Turboのテンプレートを使用する。Qwen-Image 2.1を選択する指定ではない。MCP側でテンプレートの環境変数を変更している場合は、実行モデルが変わる。

`wait=false`は生成を投入して`prompt_id`を返す動作。ツールの`Done`だけでは画像生成完了を意味しない。今回は1回のツール呼び出しとIDの返却を画面で確認した。

## 7. 生成完了と出力を確認する

同じチャットで、返された実際のIDを指定する。`<返されたID>`は前の実行結果に置き換える。

```text
ComfyUI MCPのjobツールで、次の生成状態を1回だけ確認してください。

action: status
prompt_id: <返されたID>

完了していれば出力画像のファイル名を報告してください。
失敗していればエラーを報告してください。
まだ実行中なら、その旨だけ報告して終了してください。
画像生成の再実行はしないでください。
```

まだ実行中なら、少し時間を置いて同じ状態確認を行う。投入済みのジョブを再生成しない。ComfyUIの生成履歴や、通常の`~/AI/ComfyUI/output`も確認する。保存先をカスタマイズしている場合は、ComfyUIの設定とジョブ出力を優先する。

2026年10月12日07:45（日本時間）ごろ、利用者から「確認できました」との報告があった。今回、出力画像のファイル名や生成画像自体はこの手順書に採取していない。

## 8. 実際に詰まった点と対処

### MCP登録したJSONと、起動で使うJSONが違っていた

最初にMCPを追記したのは、次の別ファイルだった。

```text
data/experimental-speed-projection/Qwen3.8-Flash-Next-experimental-speed-projection.json
```

一方、起動コマンドは`experiment-speed-projection-vision.json`を参照していた。Strata本体は起動したが、MCP登録の行が出なかった。

実際のVision用JSONへMCP設定を反映し、同じJSONを指定して再起動したところ、MCP登録の行が出た。再現手順の第4節では最初から正しいJSONを更新する。

### テンプレート探索が続き、32Kを使い切った

画像生成を広く依頼すると、テンプレート取得などを繰り返し、会話とツール結果が蓄積した。最初のMonitor画面では次を確認した。

| 項目 | 表示 |
| --- | --- |
| 最後の入力 | 32,084 tokens |
| 最後の出力 | 676 tokens |
| 合計 | 32,760 tokens |
| ステータス | `Max tokens` |
| Context fill | 100%、約33K / 32K |
| 処理状態 | `Idle` |

続けて同じ会話へ送信した際、明確なエラーが出た。

```text
prompt (32778 tokens) leaves no room to answer in the context (32768); requests are never truncated
```

これはコンテキスト上限により次の応答を開始できないエラー。今回の停止をVRAM不足と判断する根拠ではない。`もしもし？`など短い追送でも、それまでの会話・ツール定義・結果が再入力されるため、解決しなかった。

対処は、32Kから64Kへ変更し、MCPの結果上限を20,000から8,000文字へ、ラウンド上限を12から6へ変更したうえで再起動。さらに新しいチャットで`generate_image`を直接1回だけ指定した。短いプロンプトの指定だけでは設定上限は変わらない。

成功時の画面では、Context fillが26%（約17K / 64K）、生成のツール呼び出しが1回だった。ツール定義等も含むため、新しいチャットでも入力はゼロに近くなるわけではない。

### VRAM表示を生成時間や自動調停の証拠にしない

成功後のMonitor画面ではVRAMが85.8 / 96GB、GPU使用率0%、状態`Idle`だった。この値はその時点のGPUメモリ表示で、画像生成のピーク値やStrata単独の使用量を測定したものではない。

チャットの`generate_image`に表示された0.8秒はツールがIDを返すまでの時間。`wait=false`なので、画像生成完了までの所要時間ではない。Monitorのtok/sもQwenの文章生成速度であり、画像生成速度ではない。

今回のMCP接続成功だけでは、StrataとComfyUIのVRAMが自動調停されるとは判断できない。大型画像・動画モデルや別モデルとの同時実行は未検証。

## 9. 次回起動とQwen-Image 2.1への進め方

次回はComfyUIを起動し、第5節のStrata起動コマンドを使う。専用venvやモデルが残っていれば、導入コマンドを毎回実行する必要はない。MCPプロセスはStrataが起動する。

Qwen-Image 2.1は、ComfyUIで成功済みのワークフローをFile → Export (API)でJSONとして保存し、その実在するパスを`run_workflow(workflow_path=..., wait=False)`へ渡す方式が次の候補。毎回ノードやテンプレートを広く探索する必要を減らせる。

**今回、Qwen-Image 2.1のMCP経由の実行は未確認。** API形式のワークフロー保存・実行は次の作業として残す。Florence-2やQwen-VLのカスタムノードを使う追加構成も、このMCP生成確認には含めていない。

必要に応じて、動作した版を後から記録する。

```bash
cd ~/AI/Strata
git rev-parse HEAD
uv pip freeze --python ~/AI/comfy-mcp/.venv/bin/python
git -C ~/AI/ComfyUI rev-parse HEAD
```

## 関連手順・公式資料

- [続編：MiniMax H3の導入・Qwen＋MCPで動画生成・フレームレビュー・再比較](minimax-h3-comfyui-qwen-mcp-ubuntu-20261015.md)

- [StrataのUbuntu初期セットアップ](strata-flash-next-ubuntu-setup.md)
- [StrataのGPU Vision＋projection追加](strata-flash-next-vision-projection-ubuntu.md)
- [SSHとtmux](linux/ssh-tmux-session.md)
- [GPU電力上限300Wの自動適用](linux/ubuntu-nvidia-power-limit-systemd.md)
- [ComfyUI](https://github.com/Comfy-Org/ComfyUI)
- [公式ComfyUI MCP：導入・ツール・環境変数](https://github.com/Comfy-Org/comfy-mcp)
- [comfy-cli：ワークスペース設定・Analytics](https://github.com/Comfy-Org/comfy-cli)
- [Strata：Tools from MCP servers・Context](https://github.com/Niko1221/Strata/blob/main/docs/DETAILS.md)
- [Strata setup.py：エンジンの--max-context設定](https://github.com/Niko1221/Strata/blob/main/setup.py)

公式資料のリンクはmainブランチを参照するため、後日内容が変わる場合がある。
