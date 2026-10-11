# Ubuntu起動時にNVIDIA GPUの電力上限を300Wへ設定する

整理日：2026年10月12日（日本時間）。RTX PRO 6000 Blackwell Workstation Editionを搭載したUbuntu 26.04 LTSで使う手順。

`sudo nvidia-smi -i 0 -pl 300` の手動適用は実機で確認済み。起動時にも同じ設定を行うため、systemdサービスを登録する。以下は案内した設定手順であり、サービス登録後・再起動後の実機確認結果はまだ記録していない。

## 1. 対象とコマンドの配置を確認

Ubuntu側のSSH端末、またはRDP先のUbuntu端末で実行する。

```bash
command -v nvidia-smi
nvidia-smi -L
```

以下の例は `nvidia-smi` が `/usr/bin/nvidia-smi` にあり、対象GPUがインデックス `0` の構成。別の配置ならサービスの `ExecStart` を合わせる。複数GPUがある場合は対象を確認し、必要に応じて `-i` にGPU UUIDを指定する。NVIDIAは再起動をまたいだ識別にはUUIDまたはPCI bus IDを推奨している。

## 2. systemdサービスを作成

以下をまとめてコピーして実行する。

```bash
sudo tee /etc/systemd/system/nvidia-power-limit.service >/dev/null <<'EOF'
[Unit]
Description=Set NVIDIA GPU power limit to 300W
After=nvidia-persistenced.service
StartLimitIntervalSec=0

[Service]
Type=oneshot
ExecStart=/usr/bin/nvidia-smi -i 0 -pl 300
RemainAfterExit=yes
Restart=on-failure
RestartSec=5s
TimeoutStartSec=30s

[Install]
WantedBy=multi-user.target
EOF
```

| 設定 | 役割 |
| --- | --- |
| `Type=oneshot` | 電力上限を設定して処理を終了する |
| `ExecStart` | GPU 0へ300W上限を適用する。システムサービスとしてroot権限で実行 |
| `RemainAfterExit=yes` | 成功後もサービスを適用済みの状態として表示する |
| `Restart=on-failure` / `RestartSec=5s` | 起動直後にGPUが準備できていないなど、適用に失敗した場合は5秒後に再試行する |
| `TimeoutStartSec=30s` | 1回の適用処理の待ち時間を制限する |
| `StartLimitIntervalSec=0` | 起動回数による再試行の打ち切りを無効にする |
| `After=nvidia-persistenced.service` | 同サービスも起動する場合は、その後に実行する。これ自体では同サービスを起動・導入しない |
| `WantedBy=multi-user.target` | enableすると通常起動時の実行対象になる |

成功した後に常駐して電力を監視するサービスではない。設定ミスなどで失敗し続ける場合も再試行するため、ログを確認して修正する。

## 3. 自動起動を有効にし、今すぐ適用

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now nvidia-power-limit.service
```

`enable --now` は次回以降の自動起動を登録し、現在の起動でもサービスを実行する。設定のために直ちにPCを再起動する必要はない。

## 4. 適用結果を確認

```bash
systemctl is-enabled nvidia-power-limit.service
systemctl status nvidia-power-limit.service --no-pager
nvidia-smi
```

成功時の目安：

- 自動起動：`enabled`
- サービス状態：`active (exited)`
- `nvidia-smi` の消費電力／上限：`○○W / 300W`

`active (exited)` は正常。設定処理が成功して終了し、適用済みの状態として記録されていることを示す。ただし現在の電力上限そのものは `nvidia-smi` でも確認する。300Wは消費電力の固定値ではなく上限なので、アイドル中の電力はそれより低くなる。

次回、作業を保存して再起動した後にも、上記の状態と300W表示を確認する。Strata・Maya・ComfyUIなどを起動する前に確認するとよい。

## 5. うまくいかない場合

```bash
journalctl -u nvidia-power-limit.service -b --no-pager -n 50
```

ログでコマンドのパス、GPU認識、電力上限の対応範囲を確認する。手動で300Wへ変更できるかも切り分ける。

```bash
sudo nvidia-smi -i 0 -pl 300
```

サービスファイルを修正した後は、再読み込みして再実行する。

```bash
sudo systemctl daemon-reload
sudo systemctl restart nvidia-power-limit.service
nvidia-smi
```

起動後に別のツールが上限を変更した場合や、ドライバの再読み込みなどで上限が戻った場合、このサービスは自動検知しない。必要なときは上の `restart` で再適用する。

## 6. 自動適用を無効にする

```bash
sudo systemctl disable --now nvidia-power-limit.service
```

これで次回以降の自動適用と、失敗時の再試行を停止する。すでに適用した300W上限は、この操作だけでは元に戻らない。別の上限へ戻す場合は `nvidia-smi -q -d POWER` でデフォルト値と対応範囲を確認し、`sudo nvidia-smi -i 0 -pl <ワット数>` で変更する。

## 関連手順・公式資料

- [UbuntuでStrata＋Flash-Nextをセットアップ・再起動後に起動](../strata-flash-next-ubuntu-setup.md)
- [WindowsからUbuntuへの移行](../RTX_PRO_6000_Windows_to_Ubuntu_20261009.md)
- [NVIDIA nvidia-smi：電力上限とGPU識別](https://docs.nvidia.com/deploy/nvidia-smi/index.html)
- [systemd.serviceマニュアル](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html)
- [systemd.serviceの公式マニュアルソース](https://github.com/systemd/systemd/blob/main/man/systemd.service.xml)
- [systemd.unitマニュアル：起動回数の制限](https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html)
