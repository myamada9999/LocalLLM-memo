# RTX PRO 6000 PCのWindowsからUbuntuへの移行手順と対策

> **2026年10月10日追記：** その後、CUDA Toolkitの手動導入とStrataのセットアップを完了し、日本語での対話・GPU推論・300W制限・再起動後のtmux起動に成功した。最新の手順は[Strata＋Flash-NextのUbuntuセットアップ](strata-flash-next-ubuntu-setup.md)を参照。以下は主に10月8日時点の移行経緯を残した記録。

Windows上のStrataセットアップ中にNortonがスクリプトを隔離したことをきっかけに、RTX PRO 6000搭載PCをUbuntuへ移行した。Ubuntuの起動、Wi-Fi接続、NVIDIA open版ドライバによるGPU認識・画面表示改善、SSH・RDP接続までは成功している。10月8日時点ではStrataのセットアップがCUDA Toolkit導入の段階で停止していた。

整理日：2026年10月9日。主な作業は10月8日。IPアドレス、ユーザー名、ホスト名、SSIDなどは省略し、コマンド中はプレースホルダーに置き換えた。

## 対象PCと移行後の状態

| 項目 | 内容 |
| --- | --- |
| PC | SOLUTION-W189-LC270K-BXX |
| GPU | NVIDIA RTX PRO 6000 Blackwell Workstation Edition、96GB |
| CPU | Intel Core Ultra 7 270K Plus |
| RAM | 購入構成128GB。Strataの確認表示は約123GB |
| 移行前 | Windows 11 Pro |
| 移行後 | Ubuntu 26.04.1 LTS。SSHログイン画面で確認 |
| NVIDIAドライバ | 595.99.02 |
| nvidia-smiのGPUメモリー表示 | 97,887 MiB |
| Ubuntuのルート領域 | /dev/nvme0n1p2。dfの表示は約3.6T |
| リモート接続 | Debian側からSSH・RDPとも接続成功 |

今回のGPUはBlackwell版である。古い「RTX 6000 Ada」などの手順をそのまま当てはめると、ドライバ選択を誤る可能性がある。

## 移行の経緯と実際の順序

1. **Windowsで初期確認を行った。** 納品後の動作確認、RemminaからのRDP接続、日本語入力、USB Wi-Fiアダプターの認識を進めた。
2. **WindowsでStrataの導入を試した。** Qwen3.8-Flash-Nextを使う準備中、Nortonが実行中の `START-HERE.bat` を隔離した。検知されたことは確認できるが、誤検知かどうかは確定していない。
3. **UbuntuのインストールUSBを作成した。** 会話ではUbuntu 24.04 LTSが候補に挙がったが、実際には26.04系を選択してISOをUSBへ書き込み、インストールを進めた。書き込みツール名やBIOS設定の細部は今回の記録にはない。
4. **Windowsが入っていたSSDへUbuntuを入れ直した。** 対象を `nvme0` と確認して、そのWindows領域を消去してインストールした。初期の「Gen5 SSDにWindowsを残し、Gen4 SSDへUbuntuを入れる」計画から変更している。インストール後のルートは `/dev/nvme0n1p2`。デバイス名とGen世代の対応は、別途モデル名で照合する。
5. **ネットワークを使える状態にした。** USB Wi-Fiが当初使えなかったが、対象アダプターの抜き差し後にWi-Fiインターフェースが現れ、GUIから接続できた。
6. **NVIDIAドライバをopen版へ切り替えた。** GPU認識の失敗と画面ちらつき・フォント表示の乱れを対処し、`nvidia-smi` の正常動作を確認した。
7. **SSH・RDPを設定した。** 両方とも接続できるとの報告があり、SSH経由のGPU確認画面も残っている。
8. **300W制限を検討し、Strataの導入を再開した。** 電力制限は適用結果・再起動後の維持を未確認。Strataは配布済みエンジンの取得とCUDA Toolkitの自動導入で停止した。

再インストール時に `nvme0` という番号だけで対象を決めない。次の表示でモデル・容量・マウント先を照合する。今回はWindows領域を消去した経緯であり、デュアルブート完成の記録ではない。

```bash
lsblk -o NAME,SIZE,MODEL,FSTYPE,MOUNTPOINTS
```

## 詰まった点と対策の早見表

| 詰まった点 | 切り分けと対応 | 結果 |
| --- | --- | --- |
| WindowsでStrataのスクリプトが隔離される | NortonがSTART-HERE.batを隔離。Ubuntuへ移行した | Ubuntuへ移行済み |
| Ubuntuで有線LANが使えない | AQC113のatlanticドライバは動作していたがNO-CARRIER。リンク状態を確認する | 実際の接続はUSB Wi-Fiで確保 |
| USB Wi-Fiが見えない | USB認識とネットワークIFの有無を区別。Realtek側は抜き差し後にファームウェア読み込みとIF作成が進んだ | GUIでWi-Fi接続成功 |
| aptで名前解決に失敗する | Wi-Fiがまだ接続されていない段階で発生。まずネットワーク接続を確保する | Wi-Fi接続後に作業を継続 |
| nvidia-smiでNo devices were found | カーネルログでRmInitAdapter failedとopen版必須のメッセージを確認。open版へ切り替えた | GPU認識成功 |
| コマンド確認でnot found | 「コマンドがない」と「GPU初期化失敗」を区別する。会話の短い報告だけでは対象コマンドを確定できない | 最終的にnvidia-smiは動作 |
| 画面ちらつき・フォント表示の乱れ | NVIDIA open版ドライバ導入後に表示が改善した | 改善済み。VRAM干渉という推測は未確定 |
| Strataのエンジン取得でHTTP 404 | 対象リリースとlatestの両方で取得に失敗。スクリプトがソースからのビルドへ進んだ | エンジンの準備は未完了 |
| Ubuntu 26.04でCUDA自動導入が停止 | 当時のStrataスクリプトの自動導入対象は22.04／24.04のみ。CUDA Toolkit 13.0の手動導入を求められた | 手動導入・再実行後の成功は未確認 |

### Wi-Fiの切り分け

USB機器として見えることと、Wi-Fiインターフェースが使えることは別の確認になる。再発時に状態を確認するコマンドは次のとおり。

```bash
lsusb
ip -br link
nmcli device status
sudo dmesg | tail -n 80
```

今回はRealtekの `rtl8xxxu` 関連を確認し、抜き差し後にWi-Fiが使えるようになった。もう一つのAIC8800D80系アダプターも接続候補にあったが、Ubuntuで両方を同時に利用できたとは記録しない。

有線の `NO-CARRIER` はリンクが成立していない状態を示す。これだけでドライバ未導入とは判断しない。

### NVIDIAドライバの切り分け

今回のカーネルログには `RmInitAdapter failed` と、NVIDIA openカーネルモジュールを必要とする旨が出ていた。Blackwell以降はopenカーネルモジュールが必要であることはNVIDIA公式資料でも確認できる。「open版」はNVIDIA公式のカーネルモジュールを指す。

診断で使うコマンド：

```bash
lspci -nnk | grep -A3 -i nvidia
ubuntu-drivers devices
cat /proc/driver/nvidia/version
modinfo nvidia | grep license
sudo dmesg | grep -iE 'NVRM|nvidia|RmInitAdapter' | tail -n 50
```

当時案内した切り替えコマンドは次のとおり。ユーザーからopen版導入後の表示改善が報告され、最終ドライバは595.99.02だった。インストールコマンド自体の実行ログは残っていない。

```bash
sudo apt update
sudo apt install nvidia-driver-595-open
sudo reboot
```

`595` は今回の環境で案内した系列である。別環境へ再利用するときは `ubuntu-drivers devices` とパッケージ候補を確認する。

再起動後の確認：

```bash
cat /proc/driver/nvidia/version
nvidia-smi
nvidia-smi --query-gpu=name,memory.total,driver_version --format=csv
```

open版なら `NVIDIA UNIX Open Kernel Module` や `Dual MIT/GPL` の表示が確認目安になる。今回の最終GPU確認結果は次のとおり。

```text
NVIDIA RTX PRO 6000 Blackwell Workstation Edition, 97887 MiB, 595.99.02
```

## SSHとRDPの設定

### SSH

Ubuntu側で案内した手順：

```bash
sudo apt install openssh-server
sudo systemctl enable --now ssh
hostname -I
```

Debianなどの操作元から接続する。実際のIPとユーザー名へ置き換える。

```bash
ssh <UBUNTU_USER>@<UBUNTU_IP>
```

SSH接続後に `nvidia-smi` と `df -h ~` を実行した画面で、Ubuntu、GPU、ルート領域を確認できた。

### RDP

Ubuntuの **設定 → システム → リモートデスクトップ** から設定する方法を案内した。Ubuntu標準のGNOMEリモートデスクトップを使う案内であり、xrdpを導入した記録はない。

| 機能 | 用途と確認点 |
| --- | --- |
| リモートログイン | リモートからログインする。案内ではTCP 3389 |
| デスクトップ共有 | ローカルのデスクトップを共有する。併用時は3390など別ポートになる場合がある |
| Remmina | RDPを選び、UbuntuのIP、設定画面のポートと認証情報を入力する |

実際に有効にした機能・ポートの細部は未記録だが、SSH・RDPとも接続成功の報告がある。再設定ではUbuntuの画面に表示された認証情報・ポートを使う。

## GPU電力制限300W

会話では「GPUの電力制限を300Wにしておこう」と検討した。設定結果と永続化はまだ確認していない。

設定と確認の手順は次のとおり。300WがMin／Maxの範囲に入ることを確認してから設定する。

```bash
nvidia-smi -i 0 -q -d POWER
sudo nvidia-smi -i 0 -pl 300
nvidia-smi -i 0 -q -d POWER
```

`-pl` はGPUの電力上限をW単位で設定する。PC全体のコンセント消費電力を300Wに制限する機能ではない。再起動後も同じ値か確認し、常時適用する場合は起動時の再設定を別途整備する。

## Strata再開時に残った課題（10月8日時点）

最後のセットアップ画面では、GPU、CPU、RAM、PCIe 5.0 x16の確認は通過していた。選択表示は次のとおり。

| 項目 | 選択表示 |
| --- | --- |
| モデル | Qwen3.8-Flash-Next |
| 量子化 | UD-IQ4_XS |
| コンテキスト | 32,768 tokens |
| KV cache | 8-bit |
| 画像入力 | off |
| Strata | v0.1.41のエンジン取得を試行 |

エンジンの取得が404になったため、ローカルでコンパイルする経路へ進んだ。しかしCUDA Toolkit 13.0が必要で、スクリプトの自動導入はUbuntu 22.04／24.04のみ対応していたため停止した。

Ubuntu 26.04自体でGPUが動かないという問題ではなく、**その時点のセットアップスクリプトの自動導入対象から外れていた**。画面ではNVIDIAのCUDAダウンロードページからToolkitを導入して再実行するよう求められている。26.04に合う導入方法の確認、Toolkit導入、Strata再実行、LLM起動が残る。

`nvidia-smi` にCUDAのバージョンが表示されても、CUDA Toolkitやコンパイラ `nvcc` の導入完了を意味しない。ビルド準備は次のコマンドでも確認する。

```bash
command -v nvcc
nvcc --version
```

## 別の環境で行った対策との区別

以前のWindows側RDPでは、Microsoftアカウント認証の問題をローカルアカウントへの切り替えで解決し、文字のぼやけはWindows再起動で改善した。半角／全角がバッククォートになる問題はAlt＋半角／全角でIMEを切り替えた。これらはWindowsでの実績で、Ubuntuの問題の対策としては扱わない。

Windowsで2個目のUSB Wi-Fiが見えなかった際は、「WIFI 6 USB」ドライブ内の `Setup.exe` を起動する案内後に認識成功の報告があった。Ubuntuの `rtl8xxxu` と抜き差しでの改善とは別の手順である。

Debian側PCで実施したWi-FiドライバのSecure Boot対策、公開鍵登録、DKMS署名設定も別件である。今回のUbuntu側NVIDIAの主要対策はopen版への切り替えだった。Ubuntuでも同じ署名作業を実施したと扱わない。

## 確認済みの到達点（10月8日時点）

- [x] WindowsからUbuntuへ再インストールした
- [x] UbuntuでWi-Fiに接続できた
- [x] NVIDIA open版導入後に画面表示が改善した
- [x] nvidia-smiでRTX PRO 6000 BlackwellとVRAMを確認した
- [x] SSH・RDPで接続できた
- [ ] 300W制限の適用結果と再起動後の維持
- [ ] CUDA Toolkit 13.0の導入とnvccの確認
- [ ] Strataエンジンの準備
- [ ] LLM起動と推論の動作確認

## 参照資料

実機の経緯は2026年10月5日から8日の会話、初期設定メモ、10月8日のGPU確認・Strata停止画面に基づく。

- [NVIDIA公式 Open Linux Kernel Modules](https://download.nvidia.com/XFree86/Linux-x86_64/580.105.08/README/kernel_open.html)
- [NVIDIA公式 nvidia-smi](https://docs.nvidia.com/deploy/nvidia-smi/index.html)
- [Ubuntu公式 OpenSSH server](https://ubuntu.com/server/docs/openssh-server/)
- [NVIDIA CUDA Downloads](https://developer.nvidia.com/cuda-downloads)

