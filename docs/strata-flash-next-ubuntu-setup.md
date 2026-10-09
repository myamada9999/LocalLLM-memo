# RTX PRO 6000でStrata＋Qwen3.8-Flash-Nextを動かす（Ubuntu）

Ubuntu 26.04 LTS上でStrata＋Qwen3.8-Flash-Next（Unsloth UD-IQ4_XS）を起動し、日本語で対話できること、GPU推論、300Wの電力制限を確認した。さらに起動スクリプトと設定を保存し、再起動後に300Wを再設定してtmux内で起動する手順も成功した。

整理日：2026年10月10日（日本時間）。作業記録と画面で確認できた内容を中心にまとめる。IPアドレス・ユーザー名・ホスト名などは省略し、パスは `~/AI` を使って表す。

## 1. 動作確認した構成

| 項目 | 内容 |
| --- | --- |
| OS | Ubuntu 26.04.1 LTS |
| GPU | NVIDIA RTX PRO 6000 Blackwell Workstation Edition、96GB |
| NVIDIAドライバ | 595.99.02、openカーネルモジュール |
| CPU | Intel Core Ultra 7 270K Plus |
| RAM | 購入構成128GB。セットアップ表示は約123GB |
| CUDA Toolkit | 13.1系を手動導入してセットアップを継続 |
| 推論エンジン | Strata |
| Strataのコミット | `fb58e0d`（実機の `git rev-parse --short HEAD`） |
| モデル | Qwen3.8-Flash-Next |
| 量子化 | Unsloth UD-IQ4_XS |
| コンテキスト | セットアップ時の選択：32,768 tokens |
| KV cache | セットアップ時の選択：8-bit |
| 画像入力 | セットアップ時の選択：off |
| ブラウザUI | Ubuntu上のFirefox、`http://127.0.0.1:8080` |
| リモート操作 | DebianのRemminaからUbuntuへRDP接続。SSHも利用可能 |

コンテキスト・KV cache・画像入力はセットアップ画面の選択記録。最終JSONの全文は採取していない。画像を読み取る機能や画像を生成する機能は今回の動作確認に含めない。

Ubuntuのインストール・GPUドライバ・ネットワークの準備は、[WindowsからUbuntuへの移行メモ](RTX_PRO_6000_Windows_to_Ubuntu_20261009.md)を参照する。

## 2. WindowsからUbuntuへ移った理由

最初はWindowsでStrataを導入した。Hugging Faceの証明書エラーの後にダウンロードは始まったが、実行中の `START-HERE.bat` をNortonが隔離した。

その後、Windowsが入っていたSSDへUbuntuをインストールし直した。Ubuntuでも初期のネットワーク認識や画面ちらつきで詰まったが、ネットワーク接続を確保し、NVIDIA open版ドライバを導入したことで先へ進めた。

Nortonの検知が誤検知だったかは確定していない。Windows側の証明書エラーについても、今回の記録から特定の回避設定を成功手順として挙げることはしない。

## 3. CUDA Toolkitを手動で導入する

Strataの配布済みエンジン取得がHTTP 404になり、ソースからビルドする経路へ進んだ。しかし、当時のスクリプトによるCUDA Toolkitの自動導入対象はUbuntu 22.04／24.04であり、26.04では停止した。

**Ubuntu 26.04上でCUDA Toolkitとビルドツールを手動で導入し、Strataのセットアップを再実行することで解決した。** 動作確認したコミットのスクリプトはCUDA 13.0以上を要求しており、今回導入したのは13.1系。

Ubuntu側の端末で実行する。

```bash
sudo apt update
apt-cache policy cuda-toolkit
sudo apt install build-essential cuda-toolkit
```

Ubuntu 26.04の `cuda-toolkit` は、参照時点で13.1系のメタパッケージだった。別のUbuntuバージョンや追加リポジトリを使う場合は、同じ名前でも導入内容が変わるため候補を確認する。

今回のCUDA配置に合わせ、セットアップを実行する端末で環境変数を設定する。

```bash
export PATH="/usr/local/cuda-13.1/bin:$PATH"
export LD_LIBRARY_PATH="/usr/local/cuda-13.1/lib64${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
command -v nvcc
nvcc --version
```

パスの `13.1` は実際に導入されたToolkitの配置に合わせる。このexportはその端末と子プロセスに適用される。ビルドツールを後から使うときは、必要に応じて再設定する。

`nvidia-smi` の「CUDA Version」だけではToolkitや `nvcc` の導入完了を確認できない。`nvcc --version` も確認する。

## 4. Strataをセットアップする

今回の作業フォルダは `~/AI/Strata`。すでにこのフォルダがある場合は、そのインストールを使う。

新規に用意する場合の例：

```bash
mkdir -p ~/AI
git clone https://github.com/Niko1221/Strata.git ~/AI/Strata
cd ~/AI/Strata
git checkout fb58e0d
```

このcheckoutは新規導入で今回のコミットへ合わせるためのもの。既存の動作中インストールを更新する手順とは区別する。

今回の選択内容を指定するコマンド例は次のとおり。オプションは動作確認したコミットの `setup.py` で確認した。対話形式のセットアップでも同じ内容を選べる。

```bash
cd ~/AI/Strata
bash ./setup.sh --setup --family unsloth --model UD-IQ4_XS \
  --context 32768 --kv int8 --vision no --port 8080 --no-start
```

`--no-start` は導入とモデルの準備まで行い、その場では起動しない指定。GPUは対象のRTX PRO 6000を選ぶ。

最終的なセットアップ画面では、以下を確認できた。

- エンジン：`~/AI/Strata/engine/strata`。このPC用にビルド済みと表示。
- モデル：UD-IQ4_XSのGGUF 3分割ファイルが取得済み。
- 準備済みモデル：`~/AI/Strata-data/packs/unsloth-ud-iq4_xs`。
- MTP draft layer：`~/AI/Strata-data/mtp/rt`。
- 起動スクリプト：`run-unsloth-ud-iq4_xs.sh`。

GGUFのファイル名は次の3つだった。

```text
Qwen3.8-Flash-Next-UD-IQ4_XS-00001-of-00003.gguf
Qwen3.8-Flash-Next-UD-IQ4_XS-00002-of-00003.gguf
Qwen3.8-Flash-Next-UD-IQ4_XS-00003-of-00003.gguf
```

## 5. 起動して日本語で対話する

300Wの電力上限を適用する。

```bash
sudo nvidia-smi -i 0 -pl 300
```

初回の動作確認では、Ubuntuの端末から生成されたスクリプトを起動した。

```bash
cd ~/AI/Strata
bash ./run-unsloth-ud-iq4_xs.sh
```

モデルの準備が終わったら、Ubuntu上のFirefoxで開く。

```text
http://127.0.0.1:8080
```

RDP越しでも、ブラウザを開く場所はUbuntu側。`127.0.0.1` はブラウザを実行しているPC自身を指すため、Debian側のブラウザで同じURLを開く操作とは異なる。

今回の確認用プロンプト：

```text
こんにちは。日本語で自己紹介してください。
```

日本語で入力・送信し、日本語の回答が返ることを確認した。回答中の自己紹介はモデル名や機能を検証する根拠にはせず、モデルの確認はセットアップ記録・設定・ログで行う。

### RDP先で日本語入力できなかったときの対処

Remminaの「すべてのキーボードイベントを取得する」を有効にしたうえで、**RDP先のUbuntuデスクトップで開いた端末**から実行した。

```bash
ibus restart
ibus engine mozc-jp
```

その後、Ubuntu上部バーに `ja` が現れ、FirefoxのStrataチャット欄へ日本語を入力できた。

Mozcの導入、入力ソースの追加、接続元にショートカットを取られる問題は、[UbuntuのRDP接続と日本語入力](linux/ubuntu-rdp-japanese-input.md)にまとめている。

## 6. GPU推論と300W制限を確認する

Strataを動かしたまま、別のUbuntu端末で監視する。

```bash
nvidia-smi -l 1
```

ブラウザから回答を生成し、GPU使用率と電力上限などを見る。監視を終了するときは、その監視端末でCtrl+Cを押す。

回答生成中の画面では次の値を確認できた。

| 項目 | 画面の値 |
| --- | --- |
| GPU使用率 | 100% |
| GPU消費電力／上限 | 299W／300W |
| GPU温度 | 45°C |
| ファン | 30% |
| VRAM全体の使用量／総量 | 66,300 MiB／97,887 MiB（約64.7／95.6 GiB） |
| StrataエンジンのVRAM使用量 | 64,576 MiB（約63.1 GiB） |
| VRAMの空き（差分） | 31,587 MiB（約30.8 GiB） |
| Strata UIの生成速度表示 | 155.7 tok/s |

GPUプロセス一覧に `engine/strata` が計算プロセスとして表示され、回答生成時のGPU使用率は100%だった。300Wの上限も画面で確認できた。

これらは一つの回答を生成していた時点の観測値。長時間の温度、長いコンテキスト、多数同時リクエストの性能を代表するベンチマークではない。VRAM全体にはGNOMEやRDPなどの使用分も含まれる。RAM・SSDの実使用量はこの画面からは測定していない。

## 7. 起動に使うファイルを確認・保存する

実機では次を実行して、起動スクリプト、設定ファイル、コミット番号を確認した。

```bash
cd ~/AI/Strata
sed -n '1,120p' run-unsloth-ud-iq4_xs.sh
ls -lh strata-*.json
git rev-parse --short HEAD
```

起動スクリプトは仮想環境のPythonで `serve/server.py` を実行し、次の設定を渡していた。

| 項目 | 実機で確認した参照先 |
| --- | --- |
| Python | `~/AI/Strata/.venv/bin/python` |
| サーバー | `~/AI/Strata/serve/server.py` |
| エンジン指定 | `--engine strata` |
| 設定ファイル | `~/AI/Strata/strata-unsloth-ud-iq4_xs.json` |
| ポート | `--port 8080` |
| ブラウザ起動 | `--open` |
| コミット番号 | `fb58e0d` |

実際の生成スクリプトには絶対パスが書かれている。ここでは個人情報を伏せるため `~` で表記した。

バックアップはUbuntu側の端末で行う。

```bash
cd ~/AI/Strata
strata_backup_dir="$HOME/AI/Strata-backup-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$strata_backup_dir"
cp -a run-unsloth-ud-iq4_xs.sh strata-unsloth-ud-iq4_xs.json "$strata_backup_dir/"
git rev-parse HEAD > "$strata_backup_dir/commit.txt"
ls -lh "$strata_backup_dir"
```

これは起動スクリプトと設定・コミット番号の保存。モデル本体、エンジン、Python仮想環境の完全バックアップではない。再起動後はSSD上に残っている同じインストールを利用する。

設定JSONには個人用のパスや、設定によってはAPIキーが入る。公開リポジトリには実機のJSON全文をそのまま載せず、必要な項目だけを匿名化して記録する。

## 8. 再起動後はtmux内で起動する

tmuxをUbuntu側へ導入する。

```bash
sudo apt install tmux
```

作業を保存してからUbuntuを再起動する。動作中のLLMも一度終了する。

```bash
sudo reboot
```

SSHまたはRDPでUbuntuへ再接続し、端末で順番に実行する。

```bash
sudo nvidia-smi -i 0 -pl 300
tmux new-session -s strata -c "$HOME/AI/Strata" 'bash ./run-unsloth-ud-iq4_xs.sh'
```

Ubuntu上のFirefoxで `http://127.0.0.1:8080` を開き、日本語の応答を確認する。この再起動後の起動手順について、成功の報告があった。

**300Wは再起動後に手動で再設定した。起動時に自動適用するサービスや、Strataの自動起動は今回まだ設定していない。**

### tmuxから離れる・戻る

| 操作 | キー／コマンド |
| --- | --- |
| Strataを動かしたまま端末から離れる | Ctrl+bを押して離し、その後d |
| セッション一覧 | `tmux ls` |
| 同じ画面に戻る | `tmux attach -t strata` |

`strata` セッションがすでにある場合は、まずattachして状態を確認する。起動済みのサーバーへ同じ起動コマンドを重ねて実行しない。

tmux内でStrataを実行している画面のCtrl+Cは、監視の終了ではなくサーバーの停止になる。監視用端末と起動用端末を区別する。

tmuxは端末の切断・デタッチに備える仕組みで、PCの再起動や電源オフをまたいで処理を続けるものではない。再起動後は上の起動手順を実行する。

詳しい操作は[SSHとtmuxのメモ](linux/ssh-tmux-session.md)を参照。

## 9. 詰まった点と対策

| 症状 | 今回の対処・確認 |
| --- | --- |
| WindowsでSTART-HERE.batをNortonが隔離 | Ubuntuへ移行してセットアップを続行 |
| Ubuntuで画面がちらつく／GPU初期化に失敗 | NVIDIA openカーネルモジュールへ切り替え。移行メモを参照 |
| Strataの配布済みエンジン取得が404 | ソースからビルドする経路へ進み、必要なビルドツール・Toolkitを導入 |
| Ubuntu 26.04でCUDA自動導入が停止 | CUDA Toolkit 13.1系を手動導入し、環境変数を設定してセットアップを再実行 |
| RDP先で日本語へ切り替わらない | Remminaのキー取り込み、UbuntuのGUI端末でIBus再起動・Mozc選択 |
| 再起動後に同じ構成で起動したい | 生成済みスクリプトとJSONを保存し、300Wを再設定してtmuxで起動 |

通常の再起動後の起動には、生成済みの `run-unsloth-ud-iq4_xs.sh` を使う。同じモデルの導入処理を毎回やり直す必要はない。セットアップを再実行して設定を変える場合は、先に現在のJSONとスクリプトを保存する。

## 確認済みの到達点

- [x] UbuntuでNVIDIA open版ドライバとGPU認識を確認
- [x] CUDA Toolkit手動導入後、Strataのエンジン準備を完了
- [x] Qwen3.8-Flash-Next UD-IQ4_XSのモデル準備を完了
- [x] UbuntuのFirefoxでStrataのチャットを利用
- [x] RDP先で日本語入力・変換・送信
- [x] 日本語の質問に日本語で回答
- [x] 回答生成時のGPU使用率100%と300W上限を確認
- [x] 起動スクリプト・設定ファイル・コミット番号を確認し、保存手順を実施
- [x] 再起動後、300W再設定＋tmuxでの起動に成功

## 関連メモ・参照資料

- [WindowsからUbuntuへの移行](RTX_PRO_6000_Windows_to_Ubuntu_20261009.md)
- [UbuntuのRDP接続と日本語入力](linux/ubuntu-rdp-japanese-input.md)
- [SSHとtmux](linux/ssh-tmux-session.md)
- [Strata公式リポジトリ](https://github.com/Niko1221/Strata)
- [動作確認したコミットのsetup.py](https://github.com/Niko1221/Strata/blob/fb58e0d/setup.py)
- [同コミットのINSTALL.md](https://github.com/Niko1221/Strata/blob/fb58e0d/docs/INSTALL.md)
- [Ubuntu 26.04のcuda-toolkitパッケージ](https://packages.ubuntu.com/resolute/cuda-toolkit)
- [NVIDIA CUDA Installation Guide for Linux](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/)
- [NVIDIA nvidia-smi](https://docs.nvidia.com/deploy/nvidia-smi/index.html)
- [tmuxのマニュアル](https://man.openbsd.org/tmux.1)
