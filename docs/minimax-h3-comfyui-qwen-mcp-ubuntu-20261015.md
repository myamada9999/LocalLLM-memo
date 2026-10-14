# RTX PRO 6000：MiniMax H3をComfyUIで動かし、Strata＋QwenからMCPで生成・レビューする

2026年10月12〜15日（日本時間）の作業記録。UbuntuのRTX PRO 6000上でH3のローカル動画生成を確認し、Strata上のQwenから既存の公式ComfyUI MCPを通じて、プロンプト編集・生成・画像レビュー・修正後の再比較まで実行した。

## 確認した到達点

| 処理 | 結果 |
| --- | --- |
| ComfyUIで手動生成 | MP4生成成功 |
| Qwenから既存JSONを実行 | 非同期投入、同じジョブの追跡、MP4取得まで成功 |
| プロンプト編集 | set_workflow_slotで131.promptを変更し、ファイル保存成功 |
| レビュー用ワークフロー | 31ノードを実機検証、エラー・警告なし |
| 出力 | 1回の生成でMP4 1本＋序盤・中盤・終盤のPNG 3枚 |
| MCP画像返却 | inline_images=trueで画像3枚の添付記録あり |
| 修正後生成 | プロンプトのみ変更し、同じシード等で再生成成功 |
| 再比較 | 両ジョブから画像6枚を再取得し、同時点同士で比較・説明を訂正 |

追加のH3専用MCPは導入せず、接続済みの**Comfy-Org/comfy-mcp**を使った。

ツール実行と出力取得は実ログで確認した。レビューの発言がすべて正しいことを証明したものではない。画像本体を含まないチャットログでは、第三者は観察の正確さを直接照合できない。

公開用にユーザー名・ホスト名・スクリーンショット・生の会話ログは掲載しない。以下の**/home/&lt;USER&gt;/**は自分の絶対パスへ置き換える。MCPに渡すのは、接続元PCではなく、RTX PRO 6000側のUbuntuのパス。

## 1. 環境と前提

| 要素 | 確認時の構成 |
| --- | --- |
| OS／GPU | Ubuntu（既存セットアップは26.04系）／RTX PRO 6000 Blackwell Workstation Edition、96GBクラス |
| 電力上限／RAM | 300W／128GB構成、ログの総RAMは約123GiB |
| 操作側 | Strata＋qwen3.8-flash-next-unsloth-ud-iq4_xs |
| 機能 | Vision、experimental speed projection ON |
| コンテキスト | Monitorに128Kの枠。今回の起動設定全文は未採取 |
| ComfyUI | 0.39.0、Git表示 v0.39.0-34-g08ff3c11 |
| frontend／templates | 1.55.14／0.11.78 |
| comfy-cli | 1.22.0 |
| ComfyUI Python／PyTorch | 3.12.15／2.14.1+cu130 |
| MCP | comfyui、stdio、39ツールを表示 |
| ローカルUI | ComfyUI 127.0.0.1:8188／Strata 127.0.0.1:8080 |

操作側のQwen3.8と、H3の入力を処理する**qwen3vl_32b_minimax_h3_nvfp4_awq**は別モデル。StrataのQwenをH3のテキストエンコーダーとして流用しているわけではない。

既存手順：

- [StrataのUbuntuセットアップ](strata-flash-next-ubuntu-setup.md)
- [Vision＋projection ON](strata-flash-next-vision-projection-ubuntu.md)
- [公式ComfyUI MCPの導入・Strataへの登録](strata-comfyui-mcp-image-generation-ubuntu-20261012.md)

本作業では、MonitorでMCP Connectedと、run_workflow・job・fetch_outputs・system_stats・free_memory・validate_workflow・list_workflow_slots・set_workflow_slot等を確認した。

以前の記事の**max_rounds: 6**は当時の単純な画像生成用。今回の多段処理にそのまま適用しない。必要な往復を収められる値へ調整する（例：32）。今回確認した実機の上限値自体は未採取。

**max_result_chars: 8000**では、48件のスロット一覧が途中で切れた。結果を再取得しても同じ範囲しか見えない場合がある。必要なアドレスを指定し、編集・検証の結果で確認する。上限を増やす場合は会話容量との兼ね合いも見る。

MCP画像→Vision入力の接続は、先行する画像生成の作業で修正済みだった。このH3作業では修正コードの差分を新たに採取・変更していない。画像添付が返ってもLLMが見られない場合は、この接続を別途確認する。

## 2. ComfyUIの状態確認と起動

Ubuntu端末で実行する。

~~~bash
nvidia-smi
pgrep -af 'ComfyUI|main.py|strata|llama-server'

cd ~/AI/ComfyUI
git status --short
git describe --tags --always
ls -d .venv venv 2>/dev/null
df -h .
~~~

当時、ComfyUIは**~/AI/ComfyUI**、venvは**.venv**、SSD空きは約2.6TB。Gitにはサンプル画像等の削除と.comfy-downloads/があった。H3基本テンプレートに必要な版を満たしていたため、変更を破棄せず、ComfyUI更新なしで進めた。

~~~bash
cd ~/AI/ComfyUI
.venv/bin/python main.py --listen 127.0.0.1 --port 8188
~~~

既に動いていれば重複起動しない。**RDP接続先のUbuntuのブラウザ**で http://127.0.0.1:8188 を開く。接続元PCで同じURLを開くと、そのPC自身へアクセスする。

## 3. H3用モデルの配置

テンプレート → Video → **MiniMax H3 T2V**を選択する。今回の対象は公開ウェイトのローカル生成用で、クラウドAPIノードではない。

| 保存先（ComfyUIからの相対パス） | ファイル |
| --- | --- |
| models/diffusion_models/ | minimax_h3_fl2va_pruned_int8_convrot.safetensors |
| models/text_encoders/ | qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors |
| models/vae/ | minimax_h3_video_vae_int8_convrot.safetensors |
| models/vae/ | minimax_h3_audio_vae_fp32.safetensors |
| models/loras/ | minimax_h3_fl2v_turbo_8step_v1.0_comfyui_bf16.safetensors |

画面表示の合計は約39.14GB。これは表示上のファイルサイズで、必要VRAM量とは異なる。モデルリポジトリ全体の取得は不要。

### Firefoxで詰まった点

「すべてダウンロード」でポップアップがブロックされた。生成モデルは1回19.5GBの完了表示があったが、同じボタンを繰り返したため、(1)付きの完了ファイルと(2)付きのダウンロード中ファイルができた。また、ブラウザに保存してもComfyUIへ自動配置されていなかった。

対処：

1. 重複して進行中のダウンロードをキャンセル。
2. 完了ファイルを対象のmodels/以下へ移動。
3. (1)等を除き、テンプレート指定の正しい名前へ変更。
4. 残りを端末から直接ダウンロード。

### 端末から取得する

既に完了したファイルは先に正しい場所へ移す。wget -cは同じ配布ファイルの途中から再開する用途。別モデルやHTMLエラーを保存したファイルへ追記する用途ではない。

~~~bash
cd ~/AI/ComfyUI
mkdir -p models/diffusion_models models/text_encoders models/vae models/loras

wget -c --tries=10 --timeout=60 \
  -P models/diffusion_models \
  https://huggingface.co/Comfy-Org/MiniMax-H3/resolve/main/diffusion_models/minimax_h3_fl2va_pruned_int8_convrot.safetensors

wget -c --tries=10 --timeout=60 \
  -P models/text_encoders \
  https://huggingface.co/Comfy-Org/MiniMax-H3/resolve/main/text_encoders/qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors

wget -c --tries=10 --timeout=60 \
  -P models/vae \
  https://huggingface.co/Comfy-Org/MiniMax-H3/resolve/main/vae/minimax_h3_video_vae_int8_convrot.safetensors

wget -c --tries=10 --timeout=60 \
  -P models/vae \
  https://huggingface.co/Comfy-Org/MiniMax-H3/resolve/main/vae/minimax_h3_audio_vae_fp32.safetensors

wget -c --tries=10 --timeout=60 \
  -P models/loras \
  https://huggingface.co/Comfy-Org/MiniMax-H3/resolve/main/loras/minimax_h3_fl2v_turbo_8step_v1.0_comfyui_bf16.safetensors
~~~

完了後、起動端末でCtrl+CしてComfyUIを再起動し、ブラウザを再読み込みする。「不足しているモデル」が解消したか確認する。

## 4. 手動生成とAPI形式の書き出し

| 項目 | 成功した設定 |
| --- | --- |
| テンプレート | video_minimax_h3_t2v |
| 入力 | 最初・最後の画像を未接続、テキスト→動画 |
| 縦横比／解像度 | 16:9／0.4MP、32の倍数 |
| 秒数指定／fps | 5.0／24 |
| 通常ステップ／Turbo | 20／false |
| シード | 757358688076805 |
| 音声経路 | 音声VAE → VAEDecodeAudio → CreateVideo |

手動生成後、MiniMax_H3_00001_.mp4とジョブの表示時間80.53秒を確認した。これは1回の画面表示であり、条件を統一したベンチマークではない。MCP待機の回数やLLMのtok/sとも混同しない。

このワークフローはフレーム長を17k＋5へ丸める。5秒指定では計画値124フレームとなり、24fpsでフレーム数／fpsは約5.17秒。指定値5.0秒と実際の再生時間の完全一致は前提にしない。厳密な再生時間はffprobe等で確認する。

**動いたH3のタブを選び**、ファイル → エクスポート（API）でJSONを保存する。

最初は名前だけをH3にして、別タブのZ-Imageをエクスポートしていた。中身にz_image_turbo_bf16、qwen_3_4b、SaveImageがあり、H3ではなかった。ファイル名だけで判断しない。

正しいAPI形式では、MiniMaxH3ImageToVideo、H3のUNET、H3用Qwen3VL、映像・音声VAE、CreateVideo、SaveVideoを確認した。元のワークフローは23ノード。

[動作済みAPI形式](workflows/minimax-h3/minimax_h3_t2v_api_v2.json)はサブグラフ由来の140:131等のIDを含む。そのままAPI実行できた。

## 5. Qwenから既存JSONを1回実行する

~~~bash
mkdir -p ~/AI/workflows ~/AI/outputs/minimax_h3
cp ~/Downloads/minimax_h3_t2v_api_v2.json ~/AI/workflows/
~~~

Strataで新しいチャットを作り、MCPツールを有効にする。以下の<USER>を自分のユーザー名へ置き換える。

~~~text
ComfyUI MCPで次のローカルH3ワークフローを1回実行してください。

workflow_path:
/home/<USER>/AI/workflows/minimax_h3_t2v_api_v2.json
out_dir:
/home/<USER>/AI/outputs/minimax_h3

1. server_infoで127.0.0.1:8188への接続を確認する。
2. system_statsでVRAMを確認する。
3. validate_workflowで検証し、失敗したらエラーを報告する。
4. 成功したらrun_workflowをwait=falseで1回だけ呼ぶ。
   現在のプロンプト・モデル・解像度・秒数を使う。
5. 返されたprompt_idを保持し、
   job(action="wait", prompt_id=そのID, timeout_seconds=20)
   で完了まで確認する。待機時間は毎回20秒。
   タイムアウトしても同じIDを確認し、生成を再投入しない。
6. completedになったらfetch_outputsで指定先へ取得する。
7. 実際のprompt_id、完了状態、MP4保存先を報告する。
   失敗時はjob(action="error", prompt_id=そのID)で詳細を報告する。
~~~

実ログではvalid: true、partner_nodes: []、spends_credits: false、completed、error: nullを確認した。取得した最初のMCP生成MP4は1,533,846バイト。

**wait=falseのツール完了は投入成功で、生成完了ではない。** timed_out: trueでrunningなら処理は続く。同じprompt_idを追跡し、重複投入しない。

最初の実ログではQwenが20秒2回の後に120秒へ、レビュー生成でも300秒へ延長した。後に「毎回20秒」と明示すると、修正後の生成で20秒待機4回を確認できた。待機の回数から正確な生成時間は算出しない。

## 6. 編集形式とレビュー用出力を用意する

今回のcomfy-cliでは、API形式は実行に使えるが、スロット編集は**nodes[]／links[]を持つ通常形式**を対象とする。レビュー用には編集可能な形式を別に作成した。

| 同梱ファイル | 用途 |
| --- | --- |
| [minimax_h3_t2v_api_v2.json](workflows/minimax-h3/minimax_h3_t2v_api_v2.json) | 動作済み23ノードのAPI形式 |
| [minimax_h3_review_mcp.json](workflows/minimax-h3/minimax_h3_review_mcp.json) | MCPで編集・実行する31ノードの通常形式 |
| [minimax_h3_review_api.json](workflows/minimax-h3/minimax_h3_review_api.json) | 同じレビュー構成のAPI形式、直接実行・比較用 |

レビュー用通常形式は名前付きwidgets_valuesを使うMCP編集用。GUI編集には元のH3テンプレート／通常保存したワークフローを使う。

追加内容：

- 元の23ノードを保持し、140:131等を通常形式の131等へ対応付け。
- デコード済み画像から最初・中央・最後の1枚ずつを選択。
- ImageFromBatch 3個、SaveImage 3個、中央・末尾の番号計算2個を追加（計8ノード）。
- 映像・音声をまとめたMP4保存経路も維持。

PNGのために別の動画を生成せず、1回の生成から取得する。計画値124フレームの場合、0・61・123番（約0.00・2.54・5.125秒）を選ぶ。中央・末尾は計画フレーム数に追従する。デコード画像が少なければ選択ノードが範囲内へ丸めるため、実際の画像数の厳密測定ではない。

| スロット | 意味 | 初期値 |
| --- | --- | --- |
| 131.prompt | H3へのプロンプト | 元の映画風シーン |
| 129.noise_seed | シード | 757358688076805 |
| 133.value | 秒数指定 | 5 |
| 115.megapixels | 解像度 | 0.4 |
| 139.value | Turbo | false |
| 137.value／138.value | 通常／Turboステップ数 | 20／8 |

set_workflow_slotは**stdout=falseで実ファイルに保存**する。stdout=trueのまま内容を返しただけでは、後のrun_workflowに変更が反映されない場合がある。

配布前にcomfy-cli 1.22.0と静的カタログfixtureで、UI→APIの入力一致とプロンプト／シード編集差分を検証した。その後、実機のvalidate_workflowで31ノード・エラー0・警告0を確認し、生成も成功した。

### リポジトリから作業用ファイルを配置する

リポジトリを~/AI/LocalLLM-memoへ取得済みなら：

~~~bash
mkdir -p ~/AI/workflows ~/AI/outputs/minimax_h3_review
cp ~/AI/LocalLLM-memo/docs/workflows/minimax-h3/minimax_h3_review_mcp.json \
   ~/AI/workflows/
~~~

コピー先はQwenが書き換える作業用。元の構成はリポジトリに残る。再コピーすると作業用のプロンプト等が置き換わるため、必要なら変更後のファイルを先に別名保存する。

## 7. プロンプト変更・生成・3フレームレビュー

~~~text
ComfyUI MCPで動画生成と画像レビューを行ってください。

workflow_path:
/home/<USER>/AI/workflows/minimax_h3_review_mcp.json
out_dir:
/home/<USER>/AI/outputs/minimax_h3_review

作りたい動画：
夕暮れの雪の森で赤いキツネ1匹が左から右へゆっくり歩く。
固定カメラ、リアルな映像、穏やかな風と雪を踏む環境音。5秒指定。

1. server_infoとsystem_statsで接続とメモリを確認する。
2. list_workflow_slotsで編集先を確認する。
3. 英語の動画プロンプトを作り、set_workflow_slotで
   131.promptへ設定する。stdout=falseでファイル保存する。
   シード・秒数・解像度・モデル・Turboは現在値を保持する。
4. validate_workflowで検証し、問題があれば報告する。
5. 成功したらrun_workflowをwait=falseで1回だけ呼ぶ。
6. 同じprompt_idをjob(action="wait", timeout_seconds=20)で
   追跡する。待機時間は毎回20秒。再投入しない。
7. 完了後、fetch_outputsをinline_images=trueで呼ぶ。
8. PNG3枚を見て被写体・背景・構図・個体の一貫性を評価する。
   ファイル名と特徴を示し、観察・推測・判断不能を区別する。
   画像がVision入力に届かなければ、その旨を報告する。
9. MP4保存先、レビュー結果、次回の改善プロンプトを報告する。
~~~

実ログでは131.promptだけ変更され、applied: ["131.prompt"]、警告なし。converted_from_ui: true、converted_node_count: 31、MP4＋PNG3枚、画像添付3件を確認した。Qwenは遮蔽・画面端・尾先・脚等を観察し、改善案を出した。

fetch_outputsは取得先でジョブIDの短縮名に変える。時点の対応は元のURLのファイル名かoutputs_by_nodeで確認する。

| 出力ノード | 元ファイルのプレフィックス |
| --- | --- |
| 2002 | H3_review_01_first |
| 2005 | H3_review_02_middle |
| 2008 | H3_review_03_last |
| 92 | MiniMax_H3（MP4） |

## 8. VRAMの扱い

Qwenが動いた状態でH3を生成できた。ただし、全モデルの常時同時常駐や自動VRAM調停の性能を確認したものではない。

| 計測時点 | ログの値 |
| --- | --- |
| 最初のMCP生成前 | 総VRAM約95.0GiB、空き約28.2GiB |
| 次のレビュー生成前 | 空き約1.64GB（十進換算） |
| free_memory後の再計測 | 29,967,384,576バイト＝約29.97GB／27.91GiB |

Qwenは空きが少ないときにfree_memoryを呼び、ComfyUIのモデルを解放した。Strata側のQwenを停止する操作ではない。応答は解放要求の受付なので、system_statsで実際の空きを再確認する。

後の再生成ではQwenが以前の状態から解放を判断した。毎回の実測に基づく運用にするなら、生成前にsystem_statsを呼ぶ。

**生成中のピークVRAM、CPUオフロード量、SSD読み出し量は未測定。** 初期値・終了後の表示・モデルファイルサイズから推定しない。必要なら別端末で記録する。

~~~bash
mkdir -p ~/AI/measurements
nvidia-smi \
  --query-gpu=timestamp,memory.used,memory.total,utilization.gpu \
  --format=csv -l 1 > ~/AI/measurements/h3_gpu.csv
~~~

生成完了後、記録側でCtrl+C。これは測定方法の例で、今回CSVを採取したという意味ではない。

## 9. 修正後の再生成と前後比較

同じStrataチャットで続ける。

~~~text
提案した改善プロンプトを131.promptへ反映し、1回生成してください。
シード・秒数・解像度・モデル・Turboは現在値を保持してください。
検証後に非同期実行し、同じprompt_idを毎回20秒の待機で追跡してください。
完了後、inline_images=trueで出力を取得してください。

前回と今回の先頭同士・中間同士・末尾同士で比較してください。
開始時の全身の見え方、終盤の余白、脚・足先、尾先の色、
背景と個体の一貫性を評価してください。
改善・悪化・判断不能を分け、ファイル名と根拠を示してください。
~~~

実機でプロンプトだけ変更して再生成し、MP4＋PNG3枚の取得に成功した。Qwenの報告では終盤の余白・尾先は改善したが、開始時の遮蔽は悪化し、脚は改善と悪化が混在した。全項目が改善したわけではない。

同じシードでも、プロンプト変更で背景や個体が再構成され得る。同一画素・同一背景・同一個体の保証ではない。

### 比較が揺れたら6枚を再取得する

以前の説明と食い違う観察があったため、生成を追加せず、前後の画像を再取得した。

~~~text
比較を確認し直してください。今回は生成を追加しません。

fetch_outputsをinline_images=trueで呼び、両ジョブを再取得してください。
改善前prompt_id: <前回の実際のID>
改善後prompt_id: <今回の実際のID>
out_dir: /home/<USER>/AI/outputs/minimax_h3_review

先頭同士・中間同士・末尾同士で比較してください。
各判断に画像のファイル名と特徴を示してください。
以前の説明と食い違う部分は訂正してください。
画像を確認できない場合は、その旨を報告してください。
~~~

画像6枚の添付記録を確認した。Qwenは「4本すべて明確」「被写体が小さい」「黒い脚が異常」等を訂正した。

同じ取得先に既存ファイルがあると、.1.png／.1.mp4等の重複回避名が追加される。新しい生成ではなく再取得コピー。URL・サイズは同じだったが、ハッシュ比較を行っていないため、バイト一致を独立検証したとは書かない。

## 10. レビューの限界と教訓

- **観察範囲を示す。** 背景が3枚で安定していても、動画全尺で揺れ・カット・ドリフトがないことまでは断定しない。
- **目測と測定を区別する。** Qwenは幅48%・余白20%等を「実測」と書いたが、画像座標を測るツール記録はない。目測の概算として扱う。
- **不明な形状を断定しない。** 腹下の暗い形状が対側の脚かノイズか判別できないなら、判断不能とする。
- **音声経路と音声検証を区別する。** 音声VAEが有効でも、ファイルの音声トラックや内容を調べた証拠ではない。内容・品質は再生、メタデータはffprobe等で確認する。
- **位置指定の意味を確認する。** 右側の余白を増やすためのRIGHT QUARTERは右端寄りになり得る。中央より少し右で停止する等、目的に合う位置を指定する。
- **自己訂正は正解の保証ではない。** 比較レビューは元画像と照合して使う。
- **解像度の説明も確認する。** ログには0.4MPを1280×720相当とした箇所があるが、720pは約0.92MP。実際の画像サイズ／MP4メタデータを確認する。

ここまでで操作・生成・画像返却・修正後の比較を確認した。動画全体の動き、音声の品質、生成中のメモリピーク、レビューの定量的な正解率は別途検証する。

## 11. 次回の起動

モデル・venv・ワークフローが残っていれば再ダウンロード不要。

1. 第2節のコマンドでComfyUIを起動。
2. 既存のVision＋MCP設定でStrataを起動。
3. MonitorのConnectedとMCPツール有効を確認。
4. 作業用JSONを指定し、接続・メモリ・検証後に1回実行。
5. 同じジョブを完了まで追跡し、inline_images=trueで取得。
6. フレームの対応と観察範囲を示してレビュー。

既存のStrata起動例（実際に使う設定ファイルを指定）：

~~~bash
cd ~/AI/Strata
.venv/bin/python serve/server.py \
  --engine strata \
  --config experiment-speed-projection-vision.json \
  --port 8080 \
  --open
~~~

## 公式資料

- [H3ローカル生成の概要](https://docs.comfy.org/tutorials/video/minimax/minimax-h3)
- [T2V／I2V／R2Vとモデル配置](https://docs.comfy.org/tutorials/video/minimax/minimax-h3-native)
- [MiniMax公式モデルカード](https://huggingface.co/MiniMaxAI/MiniMax-H3)
- [ComfyUI用モデル](https://huggingface.co/Comfy-Org/MiniMax-H3)
- [公式ComfyUI MCP](https://github.com/Comfy-Org/comfy-mcp)
- [comfy-cli](https://github.com/Comfy-Org/comfy-cli)
- [ImageFromBatch等の実装](https://github.com/Comfy-Org/ComfyUI/blob/master/comfy_extras/nodes_images.py)
- [StrataのMCP・設定説明](https://github.com/Niko1221/Strata/blob/main/docs/DETAILS.md)

リンクのmain／masterや配布ファイルは後日変わり得る。版を固定して再現する場合はGitコミットとvenvのパッケージ一覧も記録する。
