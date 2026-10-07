# GPUワークステーション 初期動作確認・RDP・USB接続メモ

作成日：2026-10-07  
版：v0.1／レビュー用  
対象：Windows GPUワークステーションを、DebianからRemmina/RDPで操作する環境

ユーザー名、メールアドレス、ホスト名、IPアドレス、SSID、MACアドレス、製造番号などの識別情報は、プレースホルダーに置き換えるか省略した。パスワードやPINの実値は記載しない。

## 1. 環境と確認状況

### 機器構成

| 項目 | 構成 |
| --- | --- |
| Windows側 | Intel Core Ultra 7 270K Plus搭載フルタワーワークステーション |
| GPU | NVIDIA RTX PRO 6000 Blackwell Workstation Edition、96GB |
| CPUメモリー | 128GB |
| OS | Windows 11 Pro |
| NVMe SSD #1 | PCIe Gen5、Windows用 |
| NVMe SSD #2 | PCIe Gen4、今後Ubuntuを導入する予定 |
| 操作用PC | Debian、RemminaのRDP接続 |

構成は購入情報に基づく。GPUは**Blackwell**として記録する。

### 今回確認できたこと

- [x] 納品後の初期動作確認を一通り実施したとの申告。
- [x] DebianからWindowsへRDP接続できた。
- [x] RDPの文字表示は、Windows再起動後にきれいになった。
- [x] Windows側IMEで日本語入力・変換できた。
- [x] **Alt＋半角／全角**で英数／日本語を切り替えられた。
- [x] 2個目のUSB無線LANアダプターが認識された。

GPU温度・負荷試験・性能値などの個別ログ、2つのWi-Fiへの同時接続結果は、この記録では未確認。

### プレースホルダー

| 表記 | 置き換える内容 |
| --- | --- |
| `<WIN_HOST>` | Windowsのコンピューター名 |
| `<WIN_USER>` | Windowsのローカルユーザー名 |
| `<WIN_IP>` | 接続先WindowsのIPアドレス |
| `<MS_ACCOUNT_EMAIL>` | Microsoftアカウントのメールアドレス |
| `<WLAN_IF_1>`、`<WLAN_IF_2>` | Windowsの無線インターフェース名 |
| `<WIFI_PROFILE_1>`、`<WIFI_PROFILE_2>` | 保存済みWi-Fi接続プロファイル名 |

コマンド例のプレースホルダーは、実行前に自分の値へ置き換える。

## 2. 初期動作確認の手順

再確認時は、次の順に見る。

1. 外観、ケーブル接続、電源投入、Windowsの起動を確認する。
2. タスクマネージャーの「パフォーマンス」でCPU、メモリー、GPUを確認する。
3. デバイスマネージャーでGPU、ネットワークアダプター、USB機器の認識を確認する。
4. 「ディスクの管理」で、購入した2台のSSDが認識されているか確認する。
5. NVIDIAドライバーとGPU情報を確認する。
6. Windowsを再起動し、RDP接続・表示・入力を再確認する。

WindowsのPowerShellでGPU情報を確認する例：

```powershell
nvidia-smi
nvidia-smi --query-gpu=name,memory.total,driver_version --format=csv
```

GPU名、GPUメモリー容量、ドライバーバージョンを購入構成と照合する。  
`nvidia-smi`による認識確認と、CUDAアプリケーションを実行する負荷試験は、それぞれ別の確認項目になる。

## 3. DebianからWindowsへのRDP接続

### 3.1 Windows側でリモートデスクトップを有効化

Windowsで操作する。

1. 「設定」→「システム」→「リモートデスクトップ」を開く。
2. リモートデスクトップをONにする。
3. 接続するユーザーにRDP接続権限があることを確認する。

**今回の事象：** Debianからpingは通ったが、TCP 3389は `Connection refused` だった。Windows側でリモートデスクトップを有効にすると、認証画面まで進むようになった。

### 3.2 接続できない場合の切り分け

Debianの端末で実行する。

```bash
WIN_IP='<WIN_IP>'
ping -c 4 "$WIN_IP"
nc -vz "$WIN_IP" 3389
```

| 結果 | 次の確認 |
| --- | --- |
| ping成功、3389接続成功 | ユーザー名・ドメイン・パスワードなどの認証を確認する |
| ping成功、3389で拒否 | WindowsのRDP有効化、サービス、待受状態を確認する |
| 3389がタイムアウト | IPアドレス、経路、Windowsファイアウォールなどを確認する |
| ping失敗 | IPアドレスやネットワークを確認する。pingだけではRDP可否を断定しない |

Windows側の確認コマンド：

```powershell
Get-Service TermService
Get-NetTCPConnection -LocalPort 3389 -State Listen
```

今回、サービスは `Running`、TCP 3389の待受も確認できた。

### 3.3 Remminaの接続情報

今回成功した、ローカルアカウントでの接続例：

| 項目 | 入力内容 |
| --- | --- |
| プロトコル | RDP |
| サーバー | `<WIN_IP>` |
| ユーザー名 | `<WIN_USER>` |
| ドメイン | `<WIN_HOST>` |
| パスワード | Windowsのローカルアカウントに設定したパスワード |

## 4. 「パスワードが正しくない」の解決記録

### 4.1 PIN・パスキー・パスワードを区別する

Windows HelloのPIN、パスキー、Microsoftアカウントのパスワードは区別して扱う。今回のRemmina接続では、パスワード認証を使った。

MicrosoftアカウントのWebサインインがパスキーやPINへ進む場合は、「その他のサインイン方法」などから**パスワードを使う方法**を選び、パスワード自体が正しいか確認した。

### 4.2 Windowsのユーザー情報を確認する

次のコマンドは**接続先WindowsのPowerShell**で実行する。

```powershell
whoami
Get-LocalUser -Name $env:USERNAME |
    Select-Object Name, Enabled, PrincipalSource
```

今回、対象ユーザーは有効で、`PrincipalSource` は `MicrosoftAccount` だった。

追加の診断として、Windows側で次も試した。

```powershell
runas "/user:MicrosoftAccount\<MS_ACCOUNT_EMAIL>" winver
```

結果はエラー1326だった。Microsoftアカウントのメールアドレスは一致し、Webではパスワード入力でサインインできたが、Windows側での認証は成功しなかった。原因の確定には至っていない。

### 4.3 今回成功した対応

**既存のWindowsユーザーをローカルアカウントへ切り替えた。**

1. Windowsの「設定」→「アカウント」→「ユーザーの情報」を開く。
2. 「ローカルアカウントでのサインインに切り替える」を選ぶ。
3. 本人確認を行う。
4. ユーザー名は、画面に表示された既存の名前をそのまま使う。
5. ローカルアカウント用のパスワードを設定する。
6. サインアウトし、そのパスワードでサインインする。
7. Remminaに、ローカルユーザー名・Windowsのコンピューター名・新しいパスワードを設定して再接続する。

**結果：RDP接続に成功した。**

別ユーザーを新規作成した手順ではなく、既存ユーザーのサインイン方式を切り替えた記録。ユーザー名を変更する必要はなかった。

## 5. RDPの解像度・文字のぼやけ

### 設定の目安

| 場所・項目 | 設定の目安 |
| --- | --- |
| Remminaの解像度 | クライアントの解像度を使用 |
| 動的な解像度更新 | ON |
| 通常のスケールモード | OFF |
| 色深度 | True colour／32 bpp |
| 品質 | 最高（最低速）／Best (slowest) |
| Debianのディスプレイ拡大率 | 今回は100% |

上の表は今回案内した設定の目安。動的解像度ON、品質「最高」、Debianの拡大率100%は、会話で設定済みと確認できた。

### 実際に改善した操作

1. 動的解像度がONでも、文字が荒く見えていた。
2. 色深度の変更を試したが、ほとんど変化がなかった。
3. Debianの拡大率は、すでに100%だった。
4. 品質も、すでに「最高」だった。
5. **Windows側を再起動したところ、文字がきれいになった。**

原因は断定できない。再発時は、設定値の確認に加えてWindowsの再起動を確認候補にする。

### フォントスムージングを確認する場所

Remminaメイン画面の「設定／Preferences」→「RDP」→「品質設定」で「最高」を選び、**フォントスムージング**のチェックを確認できる。

「最高」の内容も全体設定で変更できる。今回のチェック状態そのものは未記録。

## 6. RDP内での日本語入力

### IMEが動くか確認する

1. Debian側を英数入力にする。
2. RDP内のWindowsでメモ帳を開く。
3. **Windows側のタスクバー**で「A」を「ひらがな／あ」に切り替える。
4. `nihongo` と入力し、スペースで「日本語」へ変換する。

この操作で日本語変換でき、Windows側のIMEが動いていることを確認できた。

### キー操作で切り替える

Remminaの「すべてのキーボードイベントを取得する」をONにする方法を案内した。続いて、半角／全角キーを押すとバッククォートが入力されることが分かった。

英語配列として解釈されている可能性があるため、**Alt＋半角／全角**を試した。

| 操作 | 今回確認した動作 |
| --- | --- |
| 半角／全角のみ | バッククォートが入力される |
| **Alt＋半角／全角** | **「A」↔「あ」が切り替わる** |
| ローマ字入力→スペース | 漢字などへ変換できる |
| Enter | 変換を確定する |

**普段の運用：RDP操作中はDebian側を英数にし、Windows側のIMEをAlt＋半角／全角で切り替える。**

Alt＋バッククォートは、Windows IMEの日本語入力ON／OFFのショートカット。今回はこれでクリック操作を省けた。半角／全角単独の配列修正は実施した記録がない。

## 7. USBポートの使い分け

付属説明書の写真で確認した表記をまとめる。

| 場所 | 形状 | 説明書の速度・数 | 用途の目安 |
| --- | --- | --- | --- |
| 前面上部 | Type-C | 20Gbps × 1 | 対応する高速外付けSSD、一時接続 |
| 前面上部 | Type-A | 5Gbps × 2 | USBメモリー、無線LANアダプター、一時接続 |
| 背面 | Type-A | 10Gbps × 2 | 対応する外付けSSDなど |
| 背面 | Type-A | 5Gbps × 4 | キーボード、マウス、無線LANアダプター、常時接続 |
| 背面 | Type-C | 存在を確認。速度は写真の本文では未確認 | 対応するType-C機器。速度は別途仕様で確認 |

- ポートの位置や説明書の表記で選ぶ。
- USB 2.0機器も、対応するType-AのUSB 3.xポートで使える。
- 転送速度は、ポート・ケーブル・接続機器それぞれの対応範囲に左右される。
- AX900／AX7はUSB 2.0接続の製品。USB 3.xポートに挿しても、製品自体がUSB 3.xになるわけではない。

## 8. USB無線LANアダプターの認識

### 確認した機器

| 機器 | 確認できた内容 |
| --- | --- |
| BrosTrend AX900 | 箱のモデル表記はAX7。Wi-Fi 6、USB 2.0、ドライバー内蔵 |
| 既存の小型アダプター | 写真にTP-Linkのロゴ。正確な型番は未確認 |
| 既存のWindows上の表示 | Realtek RTL8188EU Wireless LAN 802.11n USB 2.0 Network Adapter |

写真のTP-Link機器とRealtek表示の対応は、今回取得できた記録では確定していない。

### 症状と切り分け

2個目のAX900を接続しても、ネットワークアダプターとして追加されないように見えた。

1. Windowsのデバイスマネージャーで「ネットワーク アダプター」を確認する。
2. 対象機器の抜き差し前後で、USB機器・ドライブの変化も確認する。
3. エクスプローラーの「PC」を開き、新しいドライブが現れていないか見る。

抜き差し比較では、RDP接続に使っているアダプターを残し、確認対象を区別する。

### 今回の解決につながった手順

1. AX900を接続する。
2. エクスプローラー→「PC」で **「WIFI 6 USB」** ドライブを開く。
3. 内部の **`Setup.exe`** を実行し、ドライバーをインストールする。
4. 完了後、デバイスマネージャーとWi-Fi設定で認識を確認する。

スクリーンショットには、ドライブ内の `Readme.pdf` と `Setup.exe` が写っていた。内蔵ドライバーを起動する案内の後、ユーザーから「認識しました」との報告があった。ドライブ文字は環境によって変わる。

**要点：初回にドライバー入りドライブとして現れる製品では、エクスプローラーも確認する。**

ドライバーを再入手する場合は、BrosTrendの**AX7用**ダウンロードページを使う。AX7とLinux向けのAX7Lなどは、型番を区別して確認する。

## 9. 2つのUSB無線LANアダプターで別々のWi-Fiに接続する

それぞれのアダプターをWindowsが無線インターフェースとして認識すれば、インターフェースごとに接続先を指定できる。

今回の実績は、2個目のアダプターの認識まで。以下は同時接続を確認するための手順で、実施結果は未記録。

1. WindowsのWi-Fi設定で、2つのインターフェースを区別する。
2. 各インターフェースで目的のWi-Fiへ接続し、接続プロファイルを保存する。
3. 接続状態を確認する。

```powershell
netsh wlan show interfaces
netsh wlan show profiles
```

保存済みプロファイルを使い、インターフェースを指定して接続する例：

```powershell
netsh wlan connect name="<WIFI_PROFILE_1>" interface="<WLAN_IF_1>"
netsh wlan connect name="<WIFI_PROFILE_2>" interface="<WLAN_IF_2>"
netsh wlan show interfaces
```

`name=` は保存済みプロファイル名、`interface=` は実際の無線インターフェース名。2本の接続で、通常の通信速度が自動的に合算されるわけではない。

## 10. トラブル別の早見表

| 症状 | 今回の対応・結果 |
| --- | --- |
| pingは通るがRDP接続が拒否される | WindowsでリモートデスクトップをONにすると認証画面まで進んだ |
| MicrosoftアカウントのパスワードがWindows側で拒否される | 既存ユーザーをローカルアカウントへ切り替え、新しいパスワードでRDP接続成功 |
| 動的解像度ONでも文字が荒い | Windows再起動後に改善 |
| 日本語入力をONにできない | Windowsタスクバーで「あ」にして、IME動作を確認 |
| 半角／全角でバッククォートが出る | Alt＋半角／全角でIME切り替え成功 |
| AX900が無線LANとして見えない | 「WIFI 6 USB」ドライブのSetup.exeを起動する案内の後、認識成功の報告 |

## 11. 参考資料

実機での結果は今回の会話と提供画像に基づく。一般的な操作・仕様の確認には、次を参照した。

- [Windowsのリモートデスクトップ有効化（Microsoft）](https://learn.microsoft.com/ja-jp/windows-server/remote/remote-desktop-services/remotepc/remote-desktop-allow-access)
- [Windowsのアカウント切り替え（Microsoft）](https://support.microsoft.com/en-US/accounts-billing/manage/change-from-a-local-account-to-a-microsoft-account-in-windows)
- [Microsoft Japanese IME：キー操作・設定](https://support.microsoft.com/en-us/windows/hardware/input-devices/microsoft-japanese-ime)
- [Remmina：RDP設定の実装](https://remmina.gitlab.io/remminadoc.gitlab.io/rdp__settings_8c_source.html)
- [BrosTrend AX900／AX7：製品仕様](https://www.brostrend.com/products/ax7)
- [BrosTrend AX7：ドライバーダウンロード](https://www.brostrend.com/pages/ax7-download)
- [netsh wlan：無線インターフェースの確認と接続（Microsoft）](https://learn.microsoft.com/ja-jp/windows-server/administration/windows-commands/netsh-wlan)
- [NVIDIA System Management Interface：nvidia-smi](https://docs.nvidia.com/deploy/nvidia-smi/index.html)
