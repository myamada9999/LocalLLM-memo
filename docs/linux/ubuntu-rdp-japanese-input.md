# UbuntuのRDP接続と日本語入力（Remmina / IBus Mozc）

Ubuntu 26.04 LTSのデスクトップへDebianのRemminaからRDP接続し、日本語入力を使うための手順と、実際に解決したトラブルをまとめる。2026年10月の作業記録。Ubuntu標準のGNOMEリモートデスクトップを使用した構成。

## 1. Ubuntu側でRDPを有効にする

Ubuntuで「設定 → システム → リモートデスクトップ」を開き、用途に合わせて機能を選ぶ。

| 機能 | 用途 | ポートの目安 |
| --- | --- | --- |
| デスクトップ共有 | ログイン済みのデスクトップを共有する。操作するにはリモート操作も許可する | 通常3389。リモートログインとの併用時は3390 |
| リモートログイン | リモートからUbuntuのログイン画面を経由して利用する | 通常3389 |

**接続先ポートとRDP用のユーザー名・パスワードは、設定画面に表示される値を使う。** RDP用の認証情報は、Ubuntuの普段のログイン情報とは別の場合がある。リモートログインでは、RDP接続後にUbuntuのユーザーでログインする。

今回、RDPの接続成功は確認済み。ただし、どちらの機能を有効にしたか・使用ポートの詳細は記録していない。

IPアドレスを確認する場合は、Ubuntuの端末で実行する。

```bash
hostname -I
```

複数表示される場合は、接続元PCから到達できるLAN側のアドレスを選ぶ。

## 2. 接続元のRemminaを設定する

Debian / Ubuntuの接続元PCで、Remminaが未導入ならインストールする。

```bash
sudo apt update
sudo apt install remmina remmina-plugin-rdp
```

Remminaで接続プロファイルを作る。

| 項目 | 設定 |
| --- | --- |
| プロトコル | RDP |
| サーバー | `<UBUNTU_IP>:<RDP_PORT>`（実際のアドレスとポートに置換） |
| ユーザー名・パスワード | Ubuntuのリモートデスクトップ設定画面にあるRDP用認証情報 |

接続後、Ubuntuのデスクトップを操作できることを確認する。

## 3. Ubuntu側にMozcを導入し、入力ソースへ追加する

ここからは、**RDP先のUbuntuデスクトップで開いた端末**を使う。接続元PCの端末や通常のSSH・tmux内では、対象のGUIセッションと環境が異なる場合がある。

Mozcが未導入なら、Ubuntu側で実行する。

```bash
sudo apt update
sudo apt install ibus-mozc
```

「設定 → キーボード → 入力ソース」で、日本語のMozcを追加する。表示名は環境によって「日本語（Mozc）」や `mozc-jp` などになる。

「Japanese」だけの項目は日本語キーボード配列を示す場合があり、かな漢字変換が有効になったことの証明にはならない。

設定画面へ直接移動するには：

```bash
gnome-control-center keyboard
```

今回、設定を開くとWi-Fiのページで「システムポリシーによりWi-Fiスキャンは阻止されます」という認証ダイアログが出た。これはMozcの設定ではないため、キャンセルしてキーボード設定へ移動した。

## 4. Remminaにキーボードを取り込ませる

RDP接続画面のツールバーにあるキーボードのアイコンから、**「すべてのキーボードイベントを取得する（Grab all keyboard events）」**を有効にする。RDP画面内をクリックしてからキーを操作する。

今回、Super+Spaceなどを押しても、接続元PCの入力方式やアプリが切り替わっていた。接続元のMozcが切り替わっても、Ubuntu側の日本語入力が有効になったとは限らない。

取り込み後も、接続元のデスクトップ環境やキー設定によって一部のショートカットが接続元に取られることがある。Ubuntu側の上部バーと、実際の入力結果で確認する。

## 5. 切り替わらないときはIBusを再起動し、Mozcを選ぶ

今回、Mozcを入力ソースに登録しても上部バーの入力表示が見当たらず、ショートカットでも切り替えられなかった。

**RDP先のUbuntuデスクトップの端末で、sudoを付けずに、次の順番で実行したところ解決した。**

```bash
ibus restart
ibus engine mozc-jp
```

IBusを再起動してから、現在の入力エンジンをMozcにする操作。今回のセッションでは、Ubuntuの再起動を行わずに日本語入力ができるようになった。

うまくいかない場合は、利用可能なエンジンと現在のエンジンを確認する。

```bash
ibus list-engine
ibus engine
```

`mozc-jp` が一覧にない場合は、パッケージの導入と入力ソースの追加を確認する。インストール直後で現在のGUIセッションに反映されない場合は、作業を保存してUbuntu側でログアウト・ログインし直す。RDPウィンドウを閉じるだけでは、Ubuntuのログアウトにならない場合がある。

`ibus engine mozc-jp` は現在のセッションの選択を変更するコマンド。次回ログイン時や再起動後も同じ状態になるかは、今回まだ確認していない。

## 6. 入力ソースの選択と、Mozcのオン・オフを区別する

| 操作 | 役割 |
| --- | --- |
| 上部バーからMozcを選ぶ | Ubuntu側の入力ソースを選ぶ |
| Super+Space | GNOMEの入力ソースを切り替える（既定設定） |
| Shift+Super+Space | GNOMEの入力ソースを逆順に切り替える（既定設定） |
| 半角／全角 | Mozc選択中に日本語入力をオン・オフする（標準のMS-IMEキーマップ） |

Superは、一般的なPCキーボードのWindowsロゴキーに相当する。

まずUbuntu側でMozcを選び、入力欄をクリックして「半角／全角」を押し、`nihongo` → Space → Enterで「日本語」が入力できるか確認する。キー配列やMozcのキーマップを変更している場合は、その設定に従う。

Alt+半角／全角を追加すれば解決するとは限らない。今回のUbuntuで成功した操作は、Remminaのキー取り込みと、IBusの再起動・Mozcの選択だった。

## 7. 実際に確認できた状態

- Ubuntu上部バーに入力表示 `ja` が現れた。
- RDP先のFirefoxで、Strataのチャット欄に日本語を入力できた。
- 日本語で自己紹介を依頼し、日本語の回答が返ることを確認した。

`ja` の表示だけで判断せず、変換・確定まで確認する。LLMの自己紹介文は、読み込んだモデル名や画像対応機能を確認する根拠には使わない。実際のモデルは起動ログ・設定で確認する。

## 症状ごとの確認先

| 症状 | 確認・対処 |
| --- | --- |
| 入力ソースにMozcがない | Ubuntu側の `ibus-mozc` と入力ソースの追加を確認 |
| Super+Spaceで接続元が切り替わる | Remminaのキーボード取り込みと、RDP画面のフォーカスを確認 |
| Mozc登録済みなのに切り替わらない | UbuntuのGUI端末で `ibus restart` → `ibus engine mozc-jp` |
| 歯車を押すとWi-Fiの認証が出る | キャンセルし、`gnome-control-center keyboard` で移動 |
| すぐに日本語を入力したい | クリップボード共有が有効なら、接続元で日本語を入力・コピーしてRDP先へ貼り付ける |

## 関連メモ・参考資料

- [RTX PRO 6000のWindowsからUbuntuへの移行手順](../RTX_PRO_6000_Windows_to_Ubuntu_20261009.md)
- [SSHとtmux：再接続・スクロール・複数端末・貼り付け](ssh-tmux-session.md)
- [Ubuntu公式：Share your desktop remotely](https://ubuntu.com/desktop/docs/en/latest/how-to/share-your-desktop-remotely/)
- [Ubuntu公式：Access a remote desktop](https://ubuntu.com/desktop/docs/en/latest/how-to/access-a-remote-desktop/)
- [GNOME公式：Use alternative keyboard layouts](https://help.gnome.org/gnome-help/keyboard-layouts.html)
- [Ubuntu 26.04のibus-mozcパッケージ](https://packages.ubuntu.com/resolute/ibus-mozc)
- [Ubuntu 26.04のibusコマンドのマニュアル](https://manpages.ubuntu.com/manpages/resolute/man1/ibus.1.html)
- [Remmina公式：Features（Grab keyboard）](https://www.remmina.org/remmina-features/)
- [Mozc公式：標準MS-IMEキーマップ](https://github.com/google/mozc/blob/master/src/data/keymap/ms-ime.tsv)
