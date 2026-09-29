# Azure上のDiscord Bot稼働環境構築 学習記録

## 目次

- [1．このリポジトリについて](#1このリポジトリについて)
- [2．構築の目的](#2構築の目的)
- [3．要件と設計方針](#3要件と設計方針)
- [4．環境と構成](#4環境と構成)
- [5．Azure VMとUbuntuの初期設定](#5azure-vmとubuntuの初期設定)
- [6．Swapの作成](#6swapの作成)
- [7．Bot担当者用Linuxユーザーの作成](#7bot担当者用linuxユーザーの作成)
- [8．SSH公開鍵の登録](#8ssh公開鍵の登録)
- [9．共同管理用グループとディレクトリ](#9共同管理用グループとディレクトリ)
- [10．VS Code Remote SSHによる接続](#10vs-code-remote-sshによる接続)
- [11．Python環境の構成](#11python環境の構成)
- [12．稼働中Botの調査](#12稼働中botの調査)
- [13．発生した問題と修正](#13発生した問題と修正)
- [14．セキュリティ上の判断](#14セキュリティ上の判断)
- [15．現在の到達点](#15現在の到達点)
- [16．未確認事項と今後の課題](#16未確認事項と今後の課題)
- [17．再構築時のチェックリスト](#17再構築時のチェックリスト)
- [18．公開リポジトリへ含めないもの](#18公開リポジトリへ含めないもの)
- [19．この構築を通して学んだこと](#19この構築を通して学んだこと)

---

## 1．このリポジトリについて

このリポジトリは，知人が自宅PCで動かしていたDiscord Botを常時稼働させるため，Microsoft Azure上にUbuntu Serverの仮想マシンを構築した際の自主学習記録である．

AzureやLinuxサーバーの経験がほとんどない状態から，生成AIや公式ドキュメントを参照し，実環境でコマンドを実行しながら構築した．そのため，結果だけでなく，次の内容も記録している．

- 実際に使用したコマンド
- 各コマンドの目的と主なオプション
- 発生したエラーと原因
- エラーの修正方法
- セキュリティ面で判断したこと
- 実施済みの事項と未確認事項
- 今後改善したい点

このREADMEは，数か月後に読み返したときにも，何を考えて何を実行したかを説明・再現できることを目標としている．
<!-- っていうかAIに聞いて作ったからこういうの作っとかないと後々困るから自己学習用なんですが，この文章もあまりに口語的すぎた元の文をAIに直してもらったやつにもっかい加筆してるし -->


---

## 2．構築の目的

従来は個人のPC上でBotを動かしていたため，PCを停止するとBotも停止していた．そこで，個人PCの稼働状況に依存しない実行環境を用意することを目的として，Azure上へ移行した．

主な目的は次のとおりである．

- 個人PCを停止してもDiscord Botを稼働させる
- 電気代怖い
- 管理者とBot担当者のLinuxアカウントを分離する
- 利用者ごとに異なるSSH鍵を使用する
- SSH秘密鍵を利用者間で共有しない
- Bot担当者へ不要なsudo権限を与えない
- Bot担当者が必要なファイルを作成・編集・削除できるようにする
- Ubuntu標準Pythonを壊さず，Bot用のPython 3.14系を利用する（仮想環境の実装）
- Visual Studio Codeからサーバー上のファイルを編集できるようにする
- 小規模なVMで運用コストを抑える

---

## 3．要件と設計方針

### 3.1 利用者の分離

Linuxユーザーを次のように分けた．

```text
azureuser(管理者・bot製作者)
├─ Azure VMとUbuntuの管理担当
├─ sudo権限あり
├─ Discord Botの担当
└─ OS全体の設定変更を担当

botadmin(bot製作者)
├─ Discord Botの担当者
├─ sudo権限なし
├─ 専用SSH鍵で接続
└─ Bot関連ファイルの管理を担当
```

友人の技術力や信頼性に問題があるわけではないが（むしろ信用に値する），信頼関係と権限設計は分けて考えた．Botの運用にOS全体の管理権限は不要と判断し，`botadmin`にはsudo権限を付与しなかった．

### 3.2 Python環境の分離

Ubuntu 24.04では，システムPythonとしてPython 3.12.3が使用されていた．システムPythonを無理にPython 3.14へ置き換えると，Ubuntuの管理機能へ影響する可能性があるため，次のように分離した．

```text
Ubuntu・システム用
└─ Python 3.12.3

Discord Bot用
└─ botadminのユーザー領域にPython 3.14.7
```

### 3.3 秘密情報の分離

次の情報は共有またはGit管理しない方針とした．

- SSH秘密鍵
- Discord Botトークン
- パスワードとパスフレーズ
- APIキー
- Webhook URL
- `.env`の実データ
- Azureの認証情報

---

## 4．環境と構成

### 4.1 確認できた環境

```text
クラウド：Microsoft Azure
OS：Ubuntu Server 24.04.5 LTS
コードネーム：noble
管理ユーザー：azureuser
Bot担当ユーザー：botadmin
共同管理グループ：botops
共同管理用ディレクトリ：/opt/discord-bots
実際のBot配置場所：/home/botadmin/challenge
システムPython：Python 3.12.3
Bot実行Python：Python 3.14.7
Bot起動ファイル：/home/botadmin/challenge/main.py
Swap：1 GiB
接続方式：SSH公開鍵認証
リモート編集：Visual Studio Code Remote SSH
```

### 4.2 現時点の構成図

```text
Windows PC
├─ 管理者
│  ├─ 管理者専用SSH秘密鍵
│  └─ VS Code Remote SSH
│
└─ Bot担当者
   ├─ 担当者専用SSH秘密鍵
   └─ VS Code Remote SSH

Microsoft Azure
└─ Ubuntu Server VM
   ├─ azureuser
   │  └─ sudo権限あり
   ├─ botadmin
   │  ├─ sudo権限なし
   │  ├─ ~/.pyenv/versions/3.14.7/
   │  └─ ~/challenge/main.py
   ├─ /opt/discord-bots/
   │  └─ 共同管理用として準備
   └─ /swapfile
      └─ 1 GiB
```

> [!IMPORTANT]
> 共同管理用に`/opt/discord-bots`を準備したが，実際のBotは`/home/botadmin/challenge`へ配置された．この差が，管理者側のVS CodeからBotが見えなかった原因の一つだった．
> しかし，トークン情報の兼ね合いもあったためファイルの位置を置き換えるといったことはせず，そのまま`botadmin`の中で動かしてもらうことにした．

---

## 5．Azure VMとUbuntuの初期設定

### 5.1 パッケージ一覧を更新

```bash
sudo apt update
```

Ubuntuが参照するパッケージ一覧を更新する．ソフトウェア本体ではなく，利用可能なパッケージ情報の更新である．

- `sudo`：管理者権限で実行する
- `apt`：Ubuntuのパッケージ管理ツール
- `update`：パッケージ一覧を更新する

### 5.2 インストール済みパッケージを更新

```bash
sudo apt upgrade -y
```

- `upgrade`：既存パッケージを更新する
- `-y`：確認へ自動で同意する

構築初期段階で実行した．運用開始後は，再起動要否や変更内容を確認してから実行する方が安全である．

### 5.3 基本ツールをインストール

```bash
sudo apt install -y python3 python3-venv python3-pip git unzip
```

導入したものは次のとおりである．

- `python3`：Ubuntu標準のPython
- `python3-venv`：Python仮想環境を作成する機能
- `python3-pip`：Pythonパッケージ管理ツール
- `git`：Gitリポジトリの取得・管理
- `unzip`：ZIPファイルの展開

### 5.4 バージョン確認

```bash
python3 --version
git --version
```

確認時点のシステムPythonは次だった．

```text
Python 3.12.3
```

### 5.5 Ubuntuバージョンの確認

```bash
lsb_release -a
```

確認結果は次のとおりだった．

```text
Distributor ID: Ubuntu
Description:    Ubuntu 24.04.5 LTS
Release:        24.04
Codename:       noble
```

`No LSB modules are available.`という表示もあったが，Ubuntuのバージョン情報は取得できていた．

---

## 6．Swapの作成

VMのメモリが小さいため，Pythonのビルドや一時的なメモリ不足へ備えて，1 GiBのSwapを作成した．

### 6.1 作成前の状態確認

```bash
free -h
swapon --show
```

- `free -h`：物理メモリとSwapの使用状況を読みやすい単位で表示する
- `swapon --show`：現在有効なSwapを表示する

### 6.2 Swapファイルの確保

```bash
sudo fallocate -l 1G /swapfile
```

- `fallocate`：指定容量のファイル領域を確保する
- `-l 1G`：容量を1 GiBに指定する
- `/swapfile`：作成先

### 6.3 権限の制限

```bash
sudo chmod 600 /swapfile
```

Swapにはプロセスのデータが書き出される可能性があるため，rootだけが読み書きできるようにした．

```text
6：所有者が読み取り・書き込み可能
0：グループは権限なし
0：その他は権限なし
```

### 6.4 Swap領域として初期化

```bash
sudo mkswap /swapfile
```

### 6.5 Swapを有効化

```bash
sudo swapon /swapfile
```

### 6.6 VM再起動後も有効にする

```bash
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

- `echo`：追記する設定行を出力する
- `|`：左側の出力を右側へ渡す
- `tee -a`：管理者権限でファイル末尾へ追記する

> [!WARNING]
> 同じコマンドを複数回実行すると，`/etc/fstab`へ同じ行が重複する．再実行前には次で確認する．

```bash
grep swapfile /etc/fstab
```

### 6.7 有効化後の確認

```bash
free -h
swapon --show
```

約1 GiBのSwapが表示されれば設定完了である．

---

## 7．Bot担当者用Linuxユーザーの作成

### 7.1 ユーザーを作成

```bash
sudo adduser --disabled-password --gecos "" botadmin
```

- `adduser`：Linuxユーザーを作成する
- `--disabled-password`：パスワードによるログインを前提としない
- `--gecos ""`：氏名などの追加情報を空欄にする
- `botadmin`：作成するユーザー名

SSH公開鍵認証を使用するため，通常のパスワードログインへ依存しない構成とした．

### 7.2 ユーザー情報を確認

```bash
id botadmin
groups botadmin
```

当初の所属グループは概ね次だった．

```text
botadmin : botadmin users
```

`sudo`グループが含まれていないことを確認した．

---

## 8．SSH公開鍵の登録

### 8.1 `.ssh`ディレクトリを作成

```bash
sudo install -d -m 700 -o botadmin -g botadmin /home/botadmin/.ssh
```

- `install -d`：ディレクトリを作成する
- `-m 700`：所有者のみ読み書き・通過可能にする
- `-o botadmin`：所有者を`botadmin`にする
- `-g botadmin`：所有グループを`botadmin`にする

### 8.2 受け取った公開鍵を確認

公開鍵は，友人側で作成した鍵ペアのうち`.pub`が付いた方だけを受け取った．秘密鍵は受け取っていない．

```bash
ls -l ~/challenge_friend.pub
head -c 20 ~/challenge_friend.pub
```

`head -c 20`で先頭を確認し，`ssh-ed25519`などから始まる公開鍵であることを確認した．

### 8.3 `authorized_keys`として配置

```bash
sudo install -m 600 -o botadmin -g botadmin \
  ~/challenge_friend.pub \
  /home/botadmin/.ssh/authorized_keys
```

- `-m 600`：所有者のみ読み書き可能
- `-o botadmin`：所有者を`botadmin`にする
- `-g botadmin`：所有グループを`botadmin`にする

### 8.4 権限を確認

```bash
sudo ls -la /home/botadmin/.ssh
```

期待する状態は次である．

```text
-rw------- botadmin botadmin ... authorized_keys
```

さらに，権限・所有者・所有グループをまとめて確認した．

```bash
sudo stat -c '%A %U %G %n' \
  /home/botadmin \
  /home/botadmin/.ssh \
  /home/botadmin/.ssh/authorized_keys
```

- `%A`：権限
- `%U`：所有ユーザー
- `%G`：所有グループ
- `%n`：パス

### 8.5 転送用ファイルを削除

```bash
rm ~/challenge_friend.pub
```

`authorized_keys`への登録を確認してから，転送用の公開鍵コピーを削除した．

---

## 9．共同管理用グループとディレクトリ

### 9.1 共同管理グループを作成

```bash
sudo groupadd botops
```

### 9.2 管理者とBot担当者を追加

```bash
sudo usermod -aG botops azureuser
sudo usermod -aG botops botadmin
```

- `usermod`：既存ユーザーを変更する
- `-G botops`：補助グループとして`botops`を指定する
- `-a`：既存の補助グループを維持したまま追加する

`-a`を付けずに`-G`だけを使うと，既存の補助グループ構成を置き換える可能性があるため，追加時は`-aG`を使用した．

### 9.3 所属を確認

```bash
groups azureuser
groups botadmin
```

グループ追加後は，既存のSSHセッションへ直ちに反映されない場合がある．一度ログアウトして再接続することで確実に反映できる．

### 9.4 共同管理ディレクトリを作成

```bash
sudo mkdir -p /opt/discord-bots
sudo chown azureuser:botops /opt/discord-bots
sudo chmod 2770 /opt/discord-bots
```

#### `mkdir -p`

親ディレクトリが必要なら作成し，対象が既に存在してもエラーにしない．

#### `chown azureuser:botops`

```text
所有ユーザー：azureuser
所有グループ：botops
```

#### `chmod 2770`

```text
2：setgid
7：所有者が読み取り・書き込み・実行可能
7：グループが読み取り・書き込み・実行可能
0：その他は権限なし
```

ディレクトリへsetgidを設定することで，配下で作成するファイルやディレクトリが`botops`グループを継承しやすくなる．

### 9.5 設定を確認

```bash
ls -ld /opt/discord-bots
```

期待する表示は概ね次である．

```text
drwxrws--- ... azureuser botops ... /opt/discord-bots
```

`s`がsetgidを表す．

### 9.6 umaskを設定

```bash
grep -qxF 'umask 0002' /home/azureuser/.profile \
  || echo 'umask 0002' | sudo tee -a /home/azureuser/.profile

grep -qxF 'umask 0002' /home/botadmin/.profile \
  || echo 'umask 0002' | sudo tee -a /home/botadmin/.profile
```

- `grep -q`：一致結果を表示せず，終了状態だけ返す
- `-x`：行全体が一致する場合だけ一致扱いにする
- `-F`：正規表現ではなく固定文字列として扱う
- `||`：左側が失敗した場合だけ右側を実行する

同じ設定行が`.profile`へ重複して追加されることを避けた．

`umask 0002`により，新規ファイルへグループ書き込み権限を残しやすくした．

> [!CAUTION]
> 将来，複数のプログラマが別々のBotを担当する場合，全員を共通の`botops`へ追加すると，全Botを相互に閲覧できる可能性がある．担当者本人と管理者だけに限定したいBotでは，Botごとの専用グループまたはACLを使用する方が適切である．

---

## 10．VS Code Remote SSHによる接続

SSH設定ファイルに接続先を登録し，Visual Studio CodeのRemote SSHから接続した．

設定例は次のとおりである．

```sshconfig
Host challenge-admin
    HostName <PUBLIC_IP>
    User azureuser
    IdentityFile C:/Users/<WINDOWS_USER>/.ssh/challenge_key.pem
    IdentitiesOnly yes
```

Bot担当者側は，`User`と`IdentityFile`を担当者自身のものへ変更する．

```sshconfig
Host challenge-bot
    HostName <PUBLIC_IP>
    User botadmin
    IdentityFile C:/Users/<WINDOWS_USER>/.ssh/<PRIVATE_KEY_NAME>
    IdentitiesOnly yes
```

### 接続

PowerShellでは次のように接続できる．

```powershell
ssh challenge-admin
```

または，Visual Studio Codeで次を選択する．

```text
Remote-SSH: Connect to Host...
```

### 切断

ターミナルだけ終了する場合は次を実行する．

```bash
exit
```

Visual Studio CodeのRemote SSH接続全体を閉じる場合は，コマンドパレットから次を選択する．

```text
Remote: Close Remote Connection
```

SSH接続を終了しても，Azure VM自体は停止しない．作業後にVMを止める場合は，Azure Portalで「停止済み（割り当て解除）」まで確認する必要がある．

---

## 11．Python環境の構成

### 11.1 システムPython

```bash
python3 --version
```

```text
Python 3.12.3
```

Ubuntu標準Pythonは変更せずに維持した．

### 11.2 Python 3.14のビルド依存パッケージ

管理者側で，Python 3.14をソースからビルドするための依存パッケージを導入した．

```bash
sudo apt update
sudo apt install -y \
  build-essential \
  curl \
  git \
  libssl-dev \
  zlib1g-dev \
  libbz2-dev \
  libreadline-dev \
  libsqlite3-dev \
  libncursesw5-dev \
  xz-utils \
  tk-dev \
  libxml2-dev \
  libxmlsec1-dev \
  libffi-dev \
  liblzma-dev \
  libgdbm-dev \
  libnss3-dev \
  uuid-dev \
  libzstd-dev
```

主な役割は次のとおりである．

- `build-essential`：GCC，makeなどのビルドツール
- `libssl-dev`：SSL/TLS機能
- `zlib1g-dev`：zlib圧縮
- `libbz2-dev`：bzip2圧縮
- `libreadline-dev`：対話シェルの入力支援
- `libsqlite3-dev`：SQLite
- `xz-utils`，`liblzma-dev`：xz/LZMA圧縮
- `tk-dev`：Tkinter
- `libffi-dev`：外部関数インターフェース
- `libzstd-dev`：Zstandard圧縮

> [!IMPORTANT]
> このコマンドだけではPython 3.14はインストールされない．これはPython 3.14をビルドするための準備である．

### 11.3 Botで使用中のPython

実行中プロセスから，Botが次のPythonを使用していることを確認した．

```text
/home/botadmin/.pyenv/versions/3.14.7/bin/python
```

したがって，Bot担当者のユーザー領域へpyenv経由でPython 3.14.7が導入されている．

ただし，管理者側の履歴だけでは，次の正確なコマンドは未確認である．

- pyenvの導入
- `pyenv install 3.14.7`
- `pyenv global`または`pyenv local`
- 仮想環境の作成
- `requirements.txt`からの依存関係導入

これらは確認できるまで，「正確な実施手順」として断定しない．

### 11.4 バージョン確認時の注意

次のコマンドは，バージョン確認ではなく詳細なモジュール読み込み情報を表示する．

```bash
python3 -v
```

バージョン確認には次を使う．

```bash
python3 -V
python3 --version
```

---

## 12．稼働中Botの調査

共同管理用の`/opt/discord-bots`を確認してもBotが見つからなかったため，実行中プロセスから配置場所を逆引きした．

### 12.1 PythonまたはDiscord関連プロセスを検索

```bash
ps auxww | grep -E '[p]ython|[d]iscord'
```

`[p]ython`のように書くことで，検索に使用した`grep`自身が結果へ表示されにくくなる．

確認されたプロセスは概ね次だった．

```text
botadmin ... /home/botadmin/.pyenv/versions/3.14.7/bin/python -u main.py
```

この出力から，次を確認できた．

- 実行ユーザーは`botadmin`
- Python 3.14.7を使用している
- 起動ファイル名は`main.py`
- `-u`付きで実行されている

`-u`は，標準入出力のバッファリングを無効化または抑制し，ログをリアルタイムに出しやすくするためのオプションである．

### 12.2 PIDと完全なコマンドを確認

```bash
pgrep -af '/home/botadmin/.pyenv/versions/3.14.7/bin/python'
```

- `-a`：完全なコマンドラインを表示する
- `-f`：実行ファイル名だけでなく，コマンドライン全体を検索する

以下では，取得したPIDを`<BOT_PID>`と表記する．

### 12.3 作業ディレクトリを確認

```bash
sudo readlink -f /proc/<BOT_PID>/cwd
```

結果は次だった．

```text
/home/botadmin/challenge
```

`/proc/PID/cwd`は，対象プロセスの現在の作業ディレクトリを指す．Botが`main.py`を相対パスで実行していたため，作業ディレクトリを調べることで配置場所を特定した．

### 12.4 起動ファイルの場所を確認

```bash
sudo readlink -f /proc/<BOT_PID>/cwd/main.py
```

```text
/home/botadmin/challenge/main.py
```

### 12.5 完全な起動コマンドを確認

```bash
sudo tr '\0' ' ' < /proc/<BOT_PID>/cmdline
echo
```

`/proc/PID/cmdline`は引数をNUL文字で区切って保持しているため，`tr`で空白へ置換した．

確認した起動コマンドは次だった．

```text
/home/botadmin/.pyenv/versions/3.14.7/bin/python -u main.py
```

> [!WARNING]
> `/proc/PID/environ`にはBotトークンなどの秘密情報が含まれる可能性があるため，不用意に表示または共有しない．

### 12.6 親プロセスを確認

```bash
ps -o pid,ppid,user,lstart,cmd -p <BOT_PID>
```

- `pid`：プロセスID
- `ppid`：親プロセスID
- `user`：実行ユーザー
- `lstart`：開始日時
- `cmd`：起動コマンド

確認結果では，Botプロセスの親プロセスIDは`1`だった．

```text
PPID = 1
```

これは，現在のSSHシェルから切り離され，init/systemd配下へ再親子付けされた状態を示す．ただし，この事実だけではsystemdサービスとして正式登録されているとは断定できない．`nohup`，`setsid`，二重forkなどでもPPID 1になる場合がある．

### 12.7 systemdサービスを検索

```bash
systemctl list-units --type=service --all \
  | grep -Ei 'bot|discord|python'
```

さらに，サービス定義ファイルの内容を検索した．

```bash
sudo grep -RilE \
  'main\.py|pyenv/versions/3\.14\.7|discord' \
  /etc/systemd/system \
  /lib/systemd/system \
  2>/dev/null
```

管理者側で確認した範囲では該当サービスを発見できなかった．したがって，正式な常時起動方式は未確認である．

---

## 13．発生した問題と修正

### 13.1 公開鍵登録コマンドでパスを連結した

#### 誤ったコマンド

```bash
sudo install -m 600 -o botadmin -g botadmin \
  /tmp/challenge_friend.pub/home/botadmin/.ssh/authorized_keys
```

コピー元とコピー先の間に空白がなかったため，2つのパスが1つのパスとして解釈された．

#### 修正後

```bash
sudo install -m 600 -o botadmin -g botadmin \
  ~/challenge_friend.pub \
  /home/botadmin/.ssh/authorized_keys
```

#### 学んだこと

`install`でファイルを配置する場合，最後の2引数はコピー元とコピー先である．長いコマンドでは，パスの区切りを確認する必要がある．

### 13.2 `/tmp`と`./tmp`を混同した

```text
/tmp   ：ルート直下の一時ディレクトリ
./tmp  ：現在のディレクトリ配下のtmp
~/tmp  ：現在ユーザーのホーム配下のtmp
```

公開鍵の転送先が想定と異なり，`/tmp`からファイルを確認できなかった．最終的には管理ユーザーのホームへ直接転送し，存在を確認してから登録した．

### 13.3 グループ名を誤入力した

```bash
sudo groupadd botpos
```

本来のグループ名は`botops`だった．設定名は構成資料内で統一する必要がある．

不要なグループが残っている場合は，利用状況を確認した上で削除できる．

```bash
getent group botpos
sudo groupdel botpos
```

### 13.4 グループ作成前にユーザーを追加した

```bash
sudo usermod -aG botops azureuser
```

`botops`を作成する前に実行したため，グループが存在しないというエラーになった．

正しい順序は次である．

```bash
sudo groupadd botops
sudo usermod -aG botops azureuser
sudo usermod -aG botops botadmin
```

### 13.5 Python 3.14の依存パッケージ導入後も3.12.3だった

Python 3.14用のビルド依存パッケージは，Python本体ではない．そのため，導入後に`python3 --version`を実行しても3.12.3のままだった．

Bot用Python 3.14.7はシステムPythonを置き換えず，`botadmin`のユーザー領域へ別途導入された．

### 13.6 VS CodeでBotファイルが見えなかった

管理者側では`/opt/discord-bots`を開いていたが，実際のBotは`/home/botadmin/challenge`に置かれていた．さらに，`/home/botadmin`は他ユーザーからのアクセスが制限されていた．

実行中プロセスの`/proc/PID/cwd`を確認することで，本当の配置場所を特定できた．

### 13.7 `python3 -v`をバージョン確認に使った

小文字の`-v`は詳細出力であり，バージョン表示ではない．

```bash
python3 --version
```

または，次を使用する．

```bash
python3 -V
```

---

## 14．セキュリティ上の判断

### 14.1 SSH秘密鍵を共有しない

各利用者が自分のSSH鍵ペアを持ち，サーバーへは公開鍵だけを登録した．管理者用秘密鍵をBot担当者へ共有していない．

### 14.2 利用者ごとにLinuxユーザーを分ける

管理者用の`azureuser`と，Bot担当者用の`botadmin`を分離した．誰がどの権限で操作するかを明確にし，同一アカウントの共有を避けた．

### 14.3 Bot担当者へsudoを付与しない

Bot担当者が必要とするのは，Botファイルの編集と実行であり，OS全体の管理ではない．そのため，sudo権限は付与しなかった．

### 14.4 SSH関連ファイルの権限を制限する

```text
/home/botadmin/.ssh：700
authorized_keys：600
```

### 14.5 共同ディレクトリを第三者から隠す

`/opt/discord-bots`へ`2770`を設定し，所有者と`botops`グループ以外からアクセスできないようにした．

### 14.6 Ubuntu標準Pythonを置換しない

システムPython 3.12.3を残し，Bot用Python 3.14.7を別環境として導入した．

### 14.7 BotトークンをGitへ登録しない

Botトークンはコードへ直接書かず，Git管理外の環境変数や設定ファイルから読み込む構成が望ましい．実際の読み込み方法は現時点では未確認である．

---

## 15．現在の到達点

### 実施・確認済み

- Azure上でUbuntu Server VMを利用している
- Ubuntuの基本パッケージを更新した
- Python，venv，pip，Git，unzipを導入した
- 1 GiBのSwapを作成した
- 管理者とBot担当者を別ユーザーにした
- Bot担当者へ専用SSH公開鍵を登録した
- Bot担当者へsudo権限を付与していない
- `botops`共同管理グループを作成した
- `/opt/discord-bots`共同管理ディレクトリを作成した
- VS Code Remote SSHを利用している
- Python 3.14.7でBotプロセスが稼働している
- 実行中プロセスからBotの配置場所を特定した
- Botが`/home/botadmin/challenge/main.py`として動作していることを確認した
- Botプロセスが現在のSSHシェルから切り離されていることを確認した

### 現時点で確認できた起動コマンド

```bash
/home/botadmin/.pyenv/versions/3.14.7/bin/python -u main.py
```

---

## 16．未確認事項と今後の課題

次の項目は，実施済みと断定できるだけの確認が取れていない，または今後整備する予定である．

### 16.1 Python環境

- pyenvを導入した正確なコマンド
- Python 3.14.7を導入した正確なコマンド
- Bot専用venvの有無
- 依存パッケージの導入方法
- `requirements.txt`または`pyproject.toml`の有無

### 16.2 Bot設定

- Discord Botトークンの読み込み方法
- `.env`などの設定ファイルの管理方法
- JSONデータの保存場所
- 複数Bot間でのデータ共有有無

### 16.3 常時起動

- 現在の正式な起動方法
- systemd，nohup，setsidなどのどれを利用しているか
- VM再起動後にBotが自動起動するか
- Bot異常終了時に再起動するか

### 16.4 運用

- ログの保存場所
- ログローテーション
- JSONデータのバックアップ
- バックアップからの復元試験
- 障害時の復旧手順
- 定期的なOS更新方法

### 16.5 アクセス制御

- 現在のBotフォルダーへ管理者が通常権限でアクセスする方法
- ACLの最終設定
- Botごとの専用グループ
- 将来プログラマが増えた場合の権限分離
- SSH接続元IPアドレスの制限

---

## 17．再構築時のチェックリスト

### Azure・Ubuntu

- [ ] Azure VMを作成した
- [ ] SSH接続できる
- [ ] `sudo apt update`を実行した
- [ ] 必要なパッケージを導入した
- [ ] Ubuntuのバージョンを記録した

### メモリ

- [ ] 物理メモリを確認した
- [ ] 既存Swapを確認した
- [ ] 必要ならSwapを作成した
- [ ] `/etc/fstab`への重複追記がない
- [ ] VM再起動後もSwapが有効である

### ユーザー・SSH

- [ ] 管理者とBot担当者を別ユーザーにした
- [ ] 利用者ごとにSSH鍵を作成した
- [ ] 秘密鍵を共有していない
- [ ] `.ssh`を`700`にした
- [ ] `authorized_keys`を`600`にした
- [ ] Bot担当者へsudoを付与していない

### ファイル権限

- [ ] 共同管理する範囲を決めた
- [ ] グループを作成した
- [ ] グループへ必要なユーザーだけを追加した
- [ ] `setgid`の必要性を検討した
- [ ] その他ユーザーのアクセスを拒否した
- [ ] 将来の担当者追加を考慮した

### Python

- [ ] システムPythonを置換していない
- [ ] Botが必要とするPythonバージョンを確認した
- [ ] Bot用Pythonを別環境へ導入した
- [ ] Bot専用venvを作成した
- [ ] 依存ライブラリを記録した

### Bot運用

- [ ] Botの配置場所を記録した
- [ ] 起動コマンドを記録した
- [ ] 実行ユーザーを確認した
- [ ] 常時起動方式を確認した
- [ ] VM再起動後の自動起動を試験した
- [ ] ログの確認方法を決めた
- [ ] バックアップと復元を試験した

---

## 18．公開リポジトリへ含めないもの

次の情報は，README，Issue，コミット履歴，スクリーンショットへ含めない．

```text
Discord Botトークン
SSH秘密鍵
秘密鍵の一部分
SSH鍵のパスフレーズ
パスワード
APIキー
Webhook URL
Azure認証情報
.envの実データ
公開する必要のないパブリックIP
個人情報
実際のセーブデータ
```

### `.gitignore`の例

```gitignore
.env
.env.*
!.env.example

*.pem
*.key
id_rsa
id_ed25519

.venv/
venv/
__pycache__/
*.pyc

*.log

# 実データ
*.sqlite3
*.db
save.json
backup/
```

> [!WARNING]
> `.gitignore`へ追加する前に一度コミットした秘密情報は，通常の削除だけではGit履歴に残る．誤ってトークンをコミットした場合は，履歴修正に加えてトークン自体を失効・再発行する必要がある．

---

## 19．この構築を通して学んだこと

### 19.1 信頼関係と権限設定は別に考える

利用者を信用していても，担当業務に不要な権限を与える必要はない．権限を必要最小限にすることで，誤操作やアカウント侵害時の影響範囲を小さくできる．

### 19.2 Linuxではファイルの場所と権限を分けて考える

同じサーバー上にファイルがあっても，開いているディレクトリやユーザー権限が異なれば見えない．今回，`/opt/discord-bots`を確認してもBotが見えなかったが，実際の配置場所は`/home/botadmin/challenge`だった．

### 19.3 実行中プロセスは`/proc`から調査できる

起動ファイルの場所が不明でも，PIDを取得し，`/proc/PID/cwd`や`/proc/PID/cmdline`を確認することで，作業ディレクトリと起動コマンドを特定できた．

### 19.4 システムPythonとアプリケーション用Pythonを分離する

Ubuntuが利用するPythonと，Botが必要とするPythonを同一視しない．システムPythonを残し，アプリケーション用Pythonを別環境へ導入することで，OSへの影響を抑えられる．

### 19.5 コマンドは成功時だけでなく失敗時にも学ぶことがある

公開鍵登録時の空白不足，絶対パスと相対パスの混同，グループ名の誤入力など，初歩的な誤りも発生した．一方，エラーを確認し，ファイルの存在や権限を一つずつ調べることで修正できた．

### 19.6 「動いている」と「運用方法が確立している」は異なる

Botプロセスが稼働していても，正式な起動方法，VM再起動後の自動起動，ログ，バックアップ，復旧手順が未確認なら，運用環境としては改善余地がある．今後はsystemd，ログ管理，バックアップを整備する．

---

## 補足

このREADMEは，構築時点で確認できた情報を基にしている．環境変更やBot追加を行った場合は，「実施済み」「未確認」「今後の課題」を更新し，推測を事実として記録しないようにする．
