# UbuntuでAIC8800D80 USB Wi-Fi 6アダプターを使う

確認日：2026-10-10

RTX PRO 6000搭載PCのUbuntuで、USB Wi-Fi 6アダプター（本体表記 Model: AX7）を導入し、接続に成功した記録。USBとしては認識されたがWi-Fiインターフェースが出なかったため、外部ドライバーをビルドし、DKMS経由で導入した。Secure Bootは有効のまま、既存の署名設定を利用できた。

## 確認環境と結果

| 項目 | 確認内容 |
| --- | --- |
| OS | Ubuntu 26.04（resolute） |
| カーネル | 7.0.0-38-generic / x86_64 |
| USBデバイス | AICSemi AIC 8800D80 |
| USB ID | 368b:8d83 |
| Secure Boot | enabled |
| ドライバー | shenmintao/aic8800d80 |
| DKMS登録 | aic8800/1.0.0 |
| 結果 | ビルド・署名・ロード・Wi-Fi接続成功 |

GPU用ドライバーではなく、PCのUSB無線LAN用ドライバーを導入する。外観やAX7という型番だけでドライバーを決めず、USB IDを確認する。別リビジョン・別カーネルでの動作は別途確認が必要。

## 1. USB認識とWi-Fi状態を確認

既存のネット接続に使っているアダプターは挿したまま、新しいアダプターを追加する。

```bash
lsusb
nmcli device status
uname -r
```

今回の新しいデバイスは次の表示だった。

```text
ID 368b:8d83 AICSemi AIC 8800D80
```

当初はWi-Fiインターフェースが既存の1つだけだった。USBの認識成功と、無線LANとしての利用可能状態は別。

## 2. DKMS・Secure Boot・ログを確認

```bash
dkms status
mokutil --sb-state
sudo journalctl -k -b --no-pager | grep -Ei 'aic|368b|8d83|firmware|verification|key was rejected' | tail -60
```

今回のDKMS一覧はNVIDIAだけで、AICドライバーの登録はなかった。ログにはAICデバイスのUSB認識があり、AICドライバーの動作や署名拒否は見られなかった。

コマンド名は **mokutil**。mockutilやmockutiは誤記。

## 3. ビルド環境とソースを準備

```bash
sudo apt update
sudo apt install git build-essential dkms linux-headers-$(uname -r)
git clone https://github.com/shenmintao/aic8800d80.git ~/aic8800d80
```

既に同名ディレクトリがある場合は、中身を確認してから利用する。上書きや削除はしない。

使用したコミットを後で追跡できるよう、導入時には次も記録しておく。

```bash
git -C ~/aic8800d80 rev-parse HEAD
```

今回の会話ログにはソースのコミットSHAは残っていない。

## 4. まずビルドだけ実施

```bash
cd ~/aic8800d80/drivers/aic8800
make -j4 2>&1 | tee ~/aic8800-build.log
tail -40 ~/aic8800-build.log
```

今回、次のモジュールが生成された。

- aic_load_fw.ko
- aic8800_fdrv.ko
- aic_zlp_quirk.ko

implicit-fallthroughなどの警告は出たが、リンクまで完了した。
「Skipping BTF generation ... due to unavailability of vmlinux」も出たが、今回のビルド・ロードを妨げなかった。ビルド成功だけでは実機の接続成功までは保証しない。

パイプ経由では最後のteeの終了コードだけを見るとmakeの失敗を見落とし得る。ログのエラーと生成結果を確認する。

## 5. DKMS経由でインストール

```bash
cd ~/aic8800d80
sudo bash ./install.sh 2>&1 | tee ~/aic8800-install.log
```

Secure Boot有効時の確認：

```text
Continue installation? (y/N):
```

今回は **y** で続行し、Secure Bootは有効のままとした。スクリプトには無効化の選択肢も案内されるが、この環境では無効化は不要だった。

ログで確認できた処理：

- ファームウェア・udevルールの配置
- DKMSへのソース登録、ビルド、インストール
- 現在のカーネルのinitramfs更新
- aic8800_fdrvとaic_load_fwのロード成功

インストーラーは既存のAICファームウェアを置き換える。別のAICアダプター用設定がある環境では、既存構成を確認してから実行する。

## 6. 署名とロード結果を確認

```bash
dkms status
modinfo -F signer aic_load_fw
modinfo -F signer aic8800_fdrv
lsmod | grep aic
nmcli device status
```

今回の確認結果：

```text
aic8800/1.0.0, 7.0.0-38-generic, x86_64: installed
```

両モジュールのsignerには、このPCに設定済みのSecure Boot署名鍵名が表示された。Secure Boot有効のままロードも成功したため、新しい鍵作成・MOK登録・再起動は不要だった。

**この成功は既存の署名環境を利用できた事例。新規Ubuntu環境でも必ず追加作業不要になるわけではない。** 署名者名の表示だけで鍵の信頼は確定せず、実際のロード成功も確認する。

新しいWi-Fiインターフェースがdisconnectedで表示された。これは認識失敗ではなく、接続先未選択の状態。

## 7. 新しいアダプターを指定して接続

nmcli device statusで新しく増えたWi-Fiインターフェース名を確認し、以下の値を置き換える。

```bash
WIFI_IF="新しいWi-Fiインターフェース名"
nmcli device wifi list ifname "$WIFI_IF"
nmcli --ask device wifi connect "接続先SSID" ifname "$WIFI_IF"
nmcli device status
```

パスワードは対話入力する。コマンドへ直接記載せず、ログやシェル履歴への露出を避ける。

新しいインターフェースがconnectedになれば接続成立。今回、ユーザーが接続成功を確認した。

2本挿しでは、接続成立だけでは実際のインターネット通信が新しいアダプターを通ったとは断定できない。必要なら新しいインターフェースを指定して確認する。

```bash
ping -I "$WIFI_IF" -c 4 1.1.1.1
```

応答がない場合はICMP制限もあり得るため、それだけでドライバー故障と判断しない。SSH/RDP利用中に既存アダプターを抜くとセッションが切れる可能性があるので、切替は接続経路を確認して行う。

## トラブル時の確認

| 症状 | 確認すること |
| --- | --- |
| lsusbに出ない | USBポート、挿し直し、カーネルログ |
| USBには出るがWi-Fiが増えない | モジュールのロード、対象USB ID、ファームウェア初期化ログ |
| Key was rejected by service | 署名とMOK登録の整合。Secure Boot有効だけを原因と決めつけない |
| disconnected | 対象インターフェースを指定してSSIDへ接続 |
| カーネル更新後に使えない | 新カーネル用ヘッダー、dkms status、再ビルド結果、署名 |

```bash
sudo journalctl -k -b --no-pager | grep -Ei 'aic|firmware|verification|key was rejected' | tail -80
```

DKMSによりカーネル更新時の自動再ビルド対象になったが、将来のカーネル互換性や再起動後の動作は今回の接続成功とは別に確認する。

## 参考

- [使用したドライバー](https://github.com/shenmintao/aic8800d80)
- [インストーラーの説明](https://github.com/shenmintao/aic8800d80/blob/main/INSTALL_SCRIPT.md)
- [Debianでの別製品のSecure Boot・DKMS署名メモ](debian-tplink-archer-t3u-nano.md)

SSID、MACアドレスを含むインターフェース名、ユーザー名、ホスト名、固有の署名鍵名は本メモでは伏せている。

