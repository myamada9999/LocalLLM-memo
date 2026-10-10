# RTX PRO 6000でProject Mayaを動かす：GLM-5.3-Flash＋Vision（Ubuntu）

Project MayaをUbuntuへ導入し、最初からVisionを有効にしてチャットと画像入力を使う手順。今回の作業では、起動成功と画像を添付して説明が返るところまで確認できた。

続編：[同じUncensored Q3_K_M GGUFをllama.cppからMayaへ移行した検証記録](glm53-flash-uncensored-gguf-llama-maya-20261010.md)。モデルと設定が異なる検証のため、通常版Visionの速度測定としては扱わない。

比較：[Mayaの通常版・Uncensored Q3_K_M・Q4_K_Mの動作と測定値](maya-glm53-model-comparison-20261011.md)。通常版の速度は未測定として区別し、Q3・Q4の同じ質問での表示値をまとめた。

## 1. Project Mayaの概要

[Project Maya](https://github.com/mw00/project-maya)はStrataを基にした、GLM-5.3-Flash用の推論エンジン・サーバー・ブラウザUI。MoE expertをVRAM・RAM・NVMe SSDへ配置して、大きなモデルをローカルで動かす。

Visionは画像を入力し、内容を説明・理解する機能。今回確認したのは画像への説明応答。

## 2. 準備

- NVIDIAドライバとCUDA Toolkitを用意する。Blackwell用のビルドには、Mayaのチェック実装上CUDA Toolkit 12.8以降が必要。
- Maya-S系は約100GB規模のモデル。保存先の高速NVMeには、追加ファイルや作業領域を考慮して120GB以上の空きを見込む。
- 起動前にStrataやComfyUIなど、GPU・RAMを使う他の処理を終了し、メモリを空ける。

`nvidia-smi` の「CUDA Version」だけではToolkitがインストール済みか判断できない。必要なら `nvcc --version` でも確認する。

既存の環境準備は[Strataのセットアップ記録](strata-flash-next-ubuntu-setup.md)と[WindowsからUbuntuへの移行記録](RTX_PRO_6000_Windows_to_Ubuntu_20261009.md)を参照。

## 3. 取得と環境チェック

以下では作業場所を `~/AI/project-maya` とする。新規に取得する場合：

```bash
mkdir -p ~/AI
cd ~/AI
git clone https://github.com/mw00/project-maya.git
cd project-maya
./maya.sh --check
```

すでに取得済みなら、そのプロジェクトのディレクトリへ移動して `./maya.sh --check` を実行する。

GPU・Toolkit・コンパイラなどのチェックが通り、`This PC can run Maya.` と案内されることを確認する。今回、このチェックを通過した。

CMakeがPATH上にない場合でも、チェックが `.venv` へ導入すると案内しているなら、その案内に従ってセットアップを続ける。

## 4. モデル名のエラーと修正

最初に `--model Maya-S` と指定したところ、次のエラーで停止した。

```text
maya.py: error: argument --model: invalid choice: 'Maya-S'
(choose from Maya-S-v2-IQ2_XXS, Maya-S24, Maya-M, Maya-L)
```

GPUやメモリ不足ではなく、CLIで受け付けるモデル名の不一致だった。**`--model Maya-S-v2-IQ2_XXS` へ修正して続行した。**

今回のCLIはMaya v1.0.30のもの。READMEの説明名「Maya-S」とCLI名が一致するとは限らない。別バージョンでは、まず次で有効な選択肢を確認する。

```bash
./maya.sh --help
```

## 5. 最初からVisionを有効にしてセットアップ

Ubuntu側の端末で実行する。

```bash
cd ~/AI/project-maya
./maya.sh --setup \
  --model Maya-S-v2-IQ2_XXS \
  --gpu 0 \
  --context 32768 \
  --port 8081
```

- GPUは `0`、コンテキストは32K、ポートは8081を指定する。
- **`--no-vision` を付けない。** 画像対応について質問されたら、有効にする選択肢を選ぶ。
- ダウンロード確認には、取得元・サイズを確認して回答する。

初回は依存関係の導入、エンジンのコンパイル、モデルのダウンロード、モデルインデックスの作成などが必要。Vision用の画像エンコーダもビルド・取得する。公式READMEではエンコーダのファイルは約1.1GBと説明されている。

ポート8081は既存のStrataの8080と分けるために指定した。ただし、ポートを分けることと、GPU・RAMに余裕を持って同時実行できることは別に確認する。

この手順の後、Mayaの起動に成功した。

## 6. ブラウザとVisionの確認

Ubuntu側のブラウザで開く。

```text
http://127.0.0.1:8081
```

RDP接続中も、ブラウザを開くのは接続先Ubuntu側。接続元PCで `127.0.0.1` を開くと、そのPC自身へ接続する。

画像をチャットに添付し、次を送信する。

```text
この画像を説明して
```

今回、Windowsの「ディスクの管理」画面を入力し、画面の説明とボリューム一覧に関する回答が返った。画像添付と説明応答まで確認できた。

容量・ドライブ名などの細かい読み取りがすべて正確かは未検証。追加確認用のプロンプト例：

```text
画像に実際に表示されているディスク番号・容量・パーティションを列挙してください。
読めない項目は推測せず「不明」としてください。
```

この追加プロンプトの結果は、今回の記録には含めない。

## 7. 次回起動の参考手順

公式READMEでは、導入済みなら再度スクリプトを実行して同じモデルを起動でき、取得済みモデルの再ダウンロードは不要と説明されている。

```bash
cd ~/AI/project-maya
./maya.sh
```

停止する場合は起動端末でCtrl+C。通常の起動には `--setup` を付けず、設定変更が必要なときに再セットアップする。

この節は公式手順に基づく参考情報。今回の会話では、停止後・OS再起動後の起動はまだ確認していない。

## 8. 確認した範囲

| 項目 | 状態 |
| --- | --- |
| 環境チェック | 通過 |
| モデル名エラー | CLI名を修正して解消 |
| チャット起動 | 成功 |
| Visionの画像添付・説明応答 | 確認 |
| 細かい文字・数値の読取精度 | 未検証 |
| 生成速度・prefill速度、メモリやSSD使用量 | 未測定 |
| API接続・Computer Use・ComfyUI連携 | 未確認 |
| OS再起動後の起動 | 未確認 |

## 参照資料

- [Project Maya公式リポジトリ・README](https://github.com/mw00/project-maya)
- [セットアップ・環境チェック実装](https://github.com/mw00/project-maya/blob/main/maya.py)
- [Linux起動ラッパー](https://github.com/mw00/project-maya/blob/main/maya.sh)
- [Strata＋Flash-NextのUbuntuセットアップ](strata-flash-next-ubuntu-setup.md)
- [WindowsからUbuntuへの移行](RTX_PRO_6000_Windows_to_Ubuntu_20261009.md)
- [UbuntuのRDP接続と日本語入力](linux/ubuntu-rdp-japanese-input.md)

公式リンクの `main` は更新されるため、再現時は手元の `--help` とモデル名を照合する。
