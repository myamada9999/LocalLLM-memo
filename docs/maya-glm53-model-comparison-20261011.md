# RTX PRO 6000：Mayaで動かしたGLM-5.3-Flashのモデル比較

検証日：2026-10-10～2026-10-11（日本時間）。

**通常版Maya-SはVisionの画像入力・説明応答に成功。Uncensored Q3_K_MとQ4_K_Mは、Maya v1.0.30で短い回答・長めの回答の生成に成功した。** 同じ書店イベントの質問では、Q3が49.4 tokens/s、Q4が34.1 tokens/sだった。

この記録は実機で試した3つの構成の比較。Maya-S24・Maya-M・Maya-Lの性能は測定していない。単発の画面表示値を整理したもので、厳密なベンチマークや回答品質の順位を示すものではない。

## 1. 共通の実機環境

| 項目 | 内容 |
| --- | --- |
| OS | Ubuntu（セットアップ記録では26.04 LTS） |
| GPU | NVIDIA RTX PRO 6000 Blackwell、VRAM公称96GB、1枚 |
| CPU | Intel Core Ultra 7 270K Plus |
| RAM | 128GB（32GB×4） |
| 保存先 | ローカルNVMe SSD |
| Project Maya | v1.0.30（各検証のUIで確認） |
| Mayaの配置 | `~/AI/project-maya` |
| 接続 | Ubuntu内のFirefox → `http://127.0.0.1:8081` |

## 2. 試したモデルと確認範囲

| 構成 | 通常版Maya-S | Uncensored Q3_K_M | Uncensored Q4_K_M |
| --- | --- | --- | --- |
| モデル | 通常版GLM-5.3-Flash、CLI名 `Maya-S-v2-IQ2_XXS` | OrcaRouter GLM-5.3-Flash-Uncensored-GGUF | Q3と同じ配布リポジトリ |
| ファイル容量の目安 | 約100GB規模 | 約153GB、4分割 | 約193GB、5分割 |
| context | 32768 | 8192 | 8192 |
| Vision | 有効、画像への説明応答を確認 | `--no-vision`、未検証 | `--no-vision`、未検証 |
| 文章生成 | 起動・応答を確認 | 短い質問・書店企画で成功 | 短い質問・書店企画で成功 |
| 速度・expert配置 | 未測定 | Monitorで確認 | Monitorで確認 |
| 自動チューニングの完了 | 記録なし | `y`を選択したがスキップ。完了していない | 今回は`n`と案内。実際の選択・保存済み設定・完了ログは未確認 |
| 検証日 | 2026-10-10 | 2026-10-10 | 2026-10-11 |

- Q3・Q4の容量は配布ページの概数。Monitorのメモリ表示とは別の指標。
- 通常版とUncensored版は量子化だけが違う構成ではない。通常版でのVision成功を、Uncensored版でも確認済みとは扱わない。
- Q3・Q4は同じ配布モデルの異なる量子化。今回の回答だけでは、品質差やUncensored化の効果を評価できない。
- 独自GGUFを指定するMayaの機能は公式では実験扱い。今回の2つの量子化では文章生成に成功した。

詳しい導入記録：

- [通常版Maya-S＋Visionのセットアップ](project-maya-ubuntu-vision-setup.md)
- [Uncensored Q3_K_Mの取得・llama.cppからMayaへの移行](glm53-flash-uncensored-gguf-llama-maya-20261010.md)

## 3. 共通のテスト質問

### テストA：短い説明

```text
日本語で、GPUとCPUの違いを3文で説明してください。
```

Q4は日本語の3文で回答した。Q3の54.2 tokens/sは、以前にも同じ質問を送った後の測定。Q4の27.9 tokens/sは起動後の1リクエスト目で、キャッシュ状態がそろっていない。

### テストB：長めの書店イベント企画

```text
小さな書店で、週末の来店者を増やすイベントを企画してください。予算は3万円、スタッフは2人、準備期間は1週間です。
性質の異なる企画を3つ挙げ、それぞれの対象客、準備内容、予算配分、当日の進め方を説明してください。最後に、実行しやすい案を1つ選び、理由を述べてください。
全体を日本語で800字程度にまとめてください。
```

Q4では新しいチャットで送信し、3案と採用案・理由を含む回答が最後まで生成された。質問はQ3と同じだが、生成内容・長さ・キャッシュ・system prompt・thinking設定などを統一したベンチマークではない。800字への適合や、提案内容の正確さ・実行性は定量評価していない。

## 4. テストAの表示値

| 指標 | Q3_K_M | Q4_K_M |
| --- | ---: | ---: |
| 生成tokens | 148 | 151 |
| 応答時間（Monitorの丸め表示） | 3秒 | 7秒 |
| decode／回答生成 | 54.2 tokens/s | 27.9 tokens/s |
| prefill／入力処理 | 47 tokens/s | 18 tokens/s |
| VRAM hit | 93.8% | 83.9% |
| VRAMのexpert | 7,657個／86.4GB | 5,977個／85.5GB |
| Pinned RAMのexpert | 4,435個／58.0GB | 6,096個／96.0GB |
| SSD onlyのexpert | 4個 | 23個 |
| FROM RAM | 4.26個/token | 16.90個/token |
| FROM SSD | 0.00個/token | 0.00個/token |
| PROMOTED | 4.29個/token | 16.71個/token |

Q4のMonitorはRequests servedが1だった。Q3にはこの54.2とは別に、初期の起動ログで154 tokens、35.44 tokens/s、expert cache 86.6% hitの記録もある。54.2だけをQ3の常時性能とみなさない。

## 5. テストBの表示値

| 指標 | Q3_K_M | Q4_K_M |
| --- | ---: | ---: |
| 生成tokens | 695 | 783 |
| 応答時間（Monitorの丸め表示） | 15秒 | 24秒 |
| decode／回答生成 | 49.4 tokens/s | 34.1 tokens/s |
| prefill／入力処理 | 307 tokens/s | 100 tokens/s |
| VRAM hit | 92.3% | 88.6% |
| VRAMのexpert | 7,651個／86.4GB | 5,979個／85.5GB |
| Pinned RAMのexpert | 4,440個／58.0GB | 6,104個／96.0GB |
| SSD onlyのexpert | 5個 | 13個 |
| FROM RAM | 5.97個/token | 11.01個/token |
| FROM SSD | 0.00個/token | 0.00個/token |
| PROMOTED | 6.00個/token | 10.87個/token |

Q4のMonitorはRequests servedが2、last requestのcontextは909／8Kと表示。8K近くまで会話を伸ばした検証ではない。各Monitorのlive表示は12,096 routed expertsだった。

Q4の表示速度34.1はQ3の49.4の約69%（約31%低い）。これは今回の2件の表示値の比率であり、一般的な性能差を保証する数字ではない。応答時間には入力処理等も含まれ、生成tokensも違うため、15秒と24秒をそのまま量子化による遅延倍率とは扱わない。

## 6. メモリと速度の解釈

### 観測できたこと

- Q4はQ3より大きな量子化で、ほぼ同じexpert用VRAM容量に収まるexpert数が少なかった。
- 書店の質問では、Q4はVRAM hitが低く、FROM RAMが多かった。Pinned RAMの表示も58.0GBから96.0GBへ増えた。
- 両モデルとも、この2種類の回答ではFROM SSDが表示上0.00だった。
- Q4は1回目27.9、2回目34.1 tokens/sだった。質問も生成長も変わっているため、この上昇をキャッシュの効果だけには帰属できない。

### 推測と限界

Mayaは各層のよく使うexpertをVRAMに保持し、RAM側のexpertをCPUで計算するか、GPUへ転送して計算するかを調整する。今回のQ4では、VRAMに置けるexpert数が減りRAMからの転送が増えたことが、速度低下に寄与したと考えられる。

ただし、計算・転送・待ち時間のprofilingや、個々の最適化を有効／無効にした比較はしていない。転送だけがボトルネックと確定したわけではない。

指標の読み方：

| 表示 | 意味・注意点 |
| --- | --- |
| decode | 出力tokenの生成速度 |
| prefill | 入力処理の速度。decodeとは別に比較する |
| VRAM hit | expertアクセスのヒット率。expertの配置比率ではない |
| VRAM／Pinned RAMのGB | expert配置用の表示。GPU全体やOS全体のメモリ使用量ではない |
| FROM RAM | RAMからPCIe経由で取り込んだexpert数/token。RAMの全expertが転送されるという意味ではない |
| FROM SSD 0.00 | 丸め表示。SSDを全く使わないという意味ではない。モデル保存・起動時読み込みにはSSDを使う |
| SSD only | SSDにのみあるexpert数。今回の質問で使われなければ生成中の読み出しは起きない |

Q4のPinned RAM 96.0GBは、実機RAM 128GBに対して大きい。ほかにOSや入力処理などのメモリも必要なので、この表示だけから他のモデル・ComfyUIを同時起動できるとは判断しない。

## 7. チューニングの実施状況

Q3の起動時にはユーザーが最適化の質問に`y`を選択した。しかし前回の記録では、GPU 0に1744MiBの使用があるため自動チューニングがスキップされ、`this PC is NOT tuned`と表示されていた。

**「yを選択した」ことと「チューニングが完了した」ことを区別する。** このときのQ3は完了済みではない。1744MiBの使用元は特定していない。

Q4はまず動作確認を優先し、最適化の質問には`n`と回答するよう案内した。実際の選択や保存済み設定を確認した起動ログは今回受け取っておらず、両モデルの有効な調整値が完全に同じとは断定できない。

Mayaの通常起動にも配置・計算配分の自動判断がある。ここで「未チューニング」と記載しているのは、明示的なcalibrationの完了が確認できていないという意味で、通常起動に最適化が一切ないという意味ではない。

最適化後の比較を行う場合は、モデルごとにcalibrationの完了と設定を確認する。今後の測定ではエンジンcommit、モデルrevision、context、thinking設定、入力、キャッシュ状態、背景負荷、複数回の結果も記録する。

## 8. Q4_K_Mの取得と今回の起動手順

### 既存のHF環境で取得

Q3の取得時に使ったvenv・認証を再利用する。Q4は約193GB分の追加ファイルとなるため、保存先の空き容量には作業用の余裕も確保する。

```bash
source ~/llm-test/hf-env/bin/activate

df -h ~/models/glm53-flash-uncensored

hf download orcarouter/GLM-5.3-Flash-Uncensored-GGUF \
  --include "*Q4_K_M*.gguf" \
  --local-dir ~/models/glm53-flash-uncensored
```

保存先は`~/models/glm53-flash-uncensored/Q4_K_M/`。Q3は別のサブフォルダーに残る。

```bash
ls -lh ~/models/glm53-flash-uncensored/Q4_K_M/*.gguf
```

`00001-of-00005`～`00005-of-00005`の5ファイルをそろえる。分割GGUFの途中のファイルだけでは起動しない。

### 先頭ファイルを指定して起動

起動中のQ3や他のモデルサーバーを、その起動端末のCtrl+Cで停止してから実行する。

```bash
cd ~/AI/project-maya

./maya.sh --setup \
  --gguf-dir "$HOME/models/glm53-flash-uncensored/Q4_K_M/GLM-5.3-Flash-Uncensored-Q4_K_M-00001-of-00005.gguf" \
  --gpu 0 \
  --context 8192 \
  --port 8081 \
  --no-vision \
  --plain
```

最初はGGUFフォルダーへpack（索引）を作るので、書き込み権限と追加の空き容量が必要。モデルの再量子化は行っていない。今回はこの案内に続いて、MayaのUIで2つの回答生成を確認した。

ブラウザは接続先Ubuntu上で開く。以前のllama-uiが残った経緯があるため、今回はFirefoxのプライベートウィンドウで`http://127.0.0.1:8081`を開いた。左側のChatの下にMonitorがある。

Q3へ戻す場合の先頭ファイルは次のとおり。context・GPU・Vision条件は同じにして指定する。

```text
~/models/glm53-flash-uncensored/Q3_K_M/GLM-5.3-Flash-Uncensored-Q3_K_M-00001-of-00004.gguf
```

## 9. 現時点の判断と未検証項目

| 項目 | 結果 |
| --- | --- |
| 通常版Maya-SのVision | 画像入力・説明応答に成功 |
| Uncensored Q3の文章生成 | 成功 |
| Uncensored Q4の文章生成 | 成功 |
| Q3・Q4の同じ質問での表示速度 | 今回はQ3が速かった |
| Q4の回答品質向上 | 未評価 |
| Uncensored版のVision | 未検証 |
| 両モデルのcalibration完了後の比較 | 未実施 |
| 8K近くの長文・長時間運転 | 未検証 |
| API・AIエージェント・ツール呼び出し | 今回の検証対象外 |
| 他のモデルやComfyUIとの同時起動 | 未検証 |

速度とメモリ余裕を重視する場合は、今回の結果ではQ3が有利だった。Q4もこの実機で文章生成に使えることは確認できたが、追加のRAM使用と速度低下に見合う品質差があるかは、実際の用途で別途評価する。

## 10. 測定の根拠と参照資料

表示値は会話中に提供されたスクリーンショットと、既存の検証記録から転記した。Q4の起動設定は案内したコマンドに基づく。Web画面のモデル名表示は量子化名まで示していないため、ファイルhash・revisionを含む独立した同一性確認は実施していない。

| テスト | Chat画像 | Monitor画像 |
| --- | --- | --- |
| Q3・A | Screenshot from 2026-10-10 19-41-13.png | Screenshot from 2026-10-10 19-41-17.png |
| Q3・B | Screenshot from 2026-10-10 19-48-59.png | Screenshot from 2026-10-10 19-49-01.png |
| Q4・A | Screenshot from 2026-10-11 08-28-05.png | Screenshot from 2026-10-11 08-28-09.png |
| Q4・B | Screenshot from 2026-10-11 08-30-54.png | Screenshot from 2026-10-11 08-30-58.png |

画像本体はこの文書には同梱していない。ユーザー名・ホスト名・認証情報を公開せず、測定値を整理した。

- [Project Maya公式README](https://github.com/mw00/project-maya)
- [Project Maya v1.0.30 README](https://github.com/mw00/project-maya/blob/v1.0.30/README.md)
- [Uncensored GGUF配布ページ](https://huggingface.co/orcarouter/GLM-5.3-Flash-Uncensored-GGUF)
- [Q4_K_Mの5分割ファイル](https://huggingface.co/orcarouter/GLM-5.3-Flash-Uncensored-GGUF/tree/main/Q4_K_M)
- [Hugging Face Hub CLI](https://huggingface.co/docs/huggingface_hub/guides/cli)

公式ページのmainやモデルrevisionは更新される。ここにある数値は検証時の構成・質問に対する結果。
