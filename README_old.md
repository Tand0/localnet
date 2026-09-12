
# wsl/ansibleを使ってローカル環境の構築に必要な環境一式を作ろう

## あぶすとらくと

- インターネットから切り離されたローカル環境にサーバをおいてシステム一式を構築することはよくあります。
- そのときに困るのは rpm, apt, pip, ansible-galaxy などのリポジトリです。
- その他 TFTP, DNS, DHCP, NTP, HTTP なども環境として用意が必要でしょう。
- そんなときには ansible!  ということで、環境一式をwslを使って構築し、PC上で動かせるようにします。
- イメージとして以下の通りです。

![イメージ](./img/002pc.jpg)

## もくじ

- [wsl/ansibleを使ってローカル環境の構築に必要な環境一式を作ろう](#wslansibleを使ってローカル環境の構築に必要な環境一式を作ろう)
    - [あぶすとらくと](#あぶすとらくと)
    - [もくじ](#もくじ)
    - [初期環境構築](#初期環境構築)
        - [wslのインストールとvenv環境の構築](#wslのインストールとvenv環境の構築)
        - [ansibleのインストール](#ansibleのインストール)
        - [ansible-vault によるパスワードの設定](#ansible-vault-によるパスワードの設定)
    - [評価環境を作る](#評価環境を作る)
        - [評価用に RHEL の shell でログインする環境を Docker で作る](#評価用に-rhel-の-shell-でログインする環境を-docker-で作る)
        - [評価用に cisco のルータのシミュレータを用意する](#評価用に-cisco-のルータのシミュレータを用意する)
    - [各種リポジトリの作成](#各種リポジトリの作成)
        - [各種リポジトリフォルダを nginx で公開する](#各種リポジトリフォルダを-nginx-で公開する)
        - [rpm のリポジトリフォルダを作りインデックスを生成する(サーバ側)](#rpm-のリポジトリフォルダを作りインデックスを生成するサーバ側)
        - [rpm のローカルリポジトリを追加する(クライアント側)](#rpm-のローカルリポジトリを追加するクライアント側)
        - [pip のリポジトリフォルダを作りインデックスを生成する(サーバ側)](#pip-のリポジトリフォルダを作りインデックスを生成するサーバ側)
        - [pip のローカルリポジトリを追加する(クライアント側側)](#pip-のローカルリポジトリを追加するクライアント側側)
        - [ansible-galaxy のリポジトリフォルダを作りインデックスを生成する(サーバ側)](#ansible-galaxy-のリポジトリフォルダを作りインデックスを生成するサーバ側)
        - [TFTP を公開する](#tftp-を公開する)
        - [NTP を公開する](#ntp-を公開する)
        - [DNSとDHCPをansibleで構築する](#dnsとdhcpをansibleで構築する)
    - [全てをまとめて実行する](#全てをまとめて実行する)
    - [Windows ファイアウォールを許可する](#windows-ファイアウォールを許可する)

## 初期環境構築

### wslのインストールとvenv環境の構築

- 最初にwslをインストールしてvenv環境を構築します。
- venv環境は、構築しないとpythonでパッケージが開発により競合したりするのですが任意です。

```shell
> wsl --install
> wsl --version
WSL バージョン: 2.6.1.0
カーネル バージョン: 6.6.87.2-1
WSLg バージョン: 1.0.66
MSRDC バージョン: 1.2.6353
Direct3D バージョン: 1.611.1-81528511
DXCore バージョン: 10.0.26100.1-240331-1435.ge-release
Windows バージョン: 10.0.26200.9278
> wsl
$ cd
$ mkdir -p ando
$ cd ando
$ sudo apt update
$ sudo apt install -y python3-venv
$ python3 -m venv .env
$ source .env/bin/activate
(.env) $ pwd
/home/t-ando/ando
```

- ここまでできたら一旦コマンドプロンプトに戻ってユーザーフォルダ直下にある `.wslconfig` をメモ帳で開きます

```shell
(.env) $ exit
> notepad %USERPROFILE%\.wslconfig
```

- WSLのポートをミラーモードに変更します。

```ini
[wsl2]
networkingMode=mirrored
```

- ミラーモードを反映させます。

```shell
(.env) $ wsl --shutdown
```

- ここまでやれば次回以降は次のコマンドで環境に入れます。

``` shell
> wsl
$ source .env/bin/activate
$ cd ~/ando
(.env) $ pwd
```

### ansibleのインストール

- 次にansibleをインストールします。

```shell
> wsl
$ source .env/bin/activate
$ cd ~/ando
(.env) $ python3 -m pip install pip
(.env) $ python3 -m pip install ansible
```

### ansible-vault によるパスワードの設定

- ansible-vault によるパスワードの設定をします。
- インストール時にパスワード入力がめんどくさいので、一つにまとめる効果があります。
- まず以下のコマンドを入力します。

```shell
$ ansible-vault create secret.yml
New Vault password:
Confirm New Vault password:
```

- ここまでするとviが開くので適当に以下のように入力します。
- これによりansibleでこれらの変数を使えるようになります。

```text
p_pass: aaa
p_pass_sudo: yyy
p_pass_enable: zzz
p_pass_local_sudo: kkk

```

- 編集したい場合は次のようにします。

```shell
$ ansible-vault edit secret.yml
Vault password:
```

- 確認したい場合は次のようにします。

```shell
$ ansible-vault view secret.yml
Vault password:
p_pass: aaa
p_pass_sudo: yyy
p_pass_enable: zzz
p_pass_local_sudo: kkk
```

- 実行時は `--ask-vault-pass` を指定することで１回のパスワード入力でまとめてパスワードが投入できます。

```shell
ansible-playbook site.yml --ask-vault-pass
```

## 評価環境を作る

### 評価用に RHEL の shell でログインする環境を Docker で作る

- オプションです。
- 評価用に RHEL の shell でログインする環境を Docker で作ります。
- 本来は必要ないです。

- docker 環境そのものをまずインストール。

```shell
(.env) $ sudo apt-get update
(.env) $ sudo apt-get install -y docker.io
(.env) $ sudo systemctl start docker
(.env) $ sudo systemctl status docker
```

- 重いので終わったら停止を忘れずに実行します。

```shell
(.env) $ sudo systemctl stop docker
(.env) $ sudo systemctl status docker
```

- 気をあらためて docker で環境を作ります

```shell
(.env) $ mkdir -p rhel-ssh
(.env) $ cd rhel-ssh/
(.env) $ vi Dockerfile
```

- [Dockerfile](./ando/rhel-ssh/Dockerfile)

- 動かしてみましょう

```shell
$ sudo docker build -t rhel-ssh .
$ sudo docker run -d -p 2222:22 --name my-rhel-ssh rhel-ssh
$ sudo docker ps -a
$ ssh user@localhost -p 2222
user@localhosts password:
$ pwd
/home/user
$ ip route show | awk "/default/ {print $3}"
172.17.0.1
$ ping 172.17.0.1
PING 172.17.0.1 (172.17.0.1) 56(84) bytes of data.
64 bytes from 172.17.0.1: icmp_seq=1 ttl=64 time=0.219 ms
64 bytes from 172.17.0.1: icmp_seq=2 ttl=64 time=0.043 ms
^C
--- 172.17.0.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4073ms
rtt min/avg/max/mdev = 0.043/0.091/0.219/0.065 ms
$ exit
logout
Connection to localhost closed.
```

- ansible で接続できるかのチェックを行います。
- まず [hosts.yml](./ando/hosts.yml) と [site_rhel_ssh.yml](./ando/site_rhel_ssh.yml) を作ります。
- コマンドとしては以下となります。成功すればOKです。

```shell
ansible-playbook site_rhel_ssh.yml -i hosts.yml --ask-vault-pass
```

- [ansible.cfg](./ando/ansible.cfg) を置くとさらに簡単に記述できます。

```shell
ansible-playbook site_rhel_ssh.yml
```

### 評価用に cisco のルータのシミュレータを用意する

- オプションです。
- cisco のルータなどを ansible-playbook で叩きたいが手元にない場合、シミュレータで動作確認します。
- そのために GNS3 を使います。
- [https://www.gns3.com/](https://www.gns3.com/) を参照してください。

## 各種リポジトリの作成

### 各種リポジトリフォルダを nginx で公開する

- 起動は [site_nginx_start.yml](./ando/site_nginx_start.yml) で実施します。
- 停止は [site_nginx_start.yml](./ando/site_nginx_start.yml) で実施します。
- どちらも `ansible-playbook site_XXXX.yml` で実行できます。
- nginx の設定ファイルは [nginx.conf.j2](./ando/nginx.conf.j2) です。
- ここでは `ando/repo` フォルダ配下を公開しています

### rpm のリポジトリフォルダを作りインデックスを生成する(サーバ側)

- 設計として `/home/t-ando/ando/rpm/rpm_repo/` の配下に作っていきます。
- ここに rpm ファイルを配置していくだけです。
- その後、リポジトリのインデックスを生成します。
    - 具体手的には [site_rpm_repo.yml](./ando/site_rpm_repo.yml) を起動します。

### rpm のローカルリポジトリを追加する(クライアント側)

- 配置後、ライアント側にこのローカルアドレスを設定するようにします。
    - 具体手的には [site_rhel_rpm.yml](./ando/site_rhel_rpm.yml) を起動します。
    - `local_repo_url` は適宜変更が必要です。

```shell
ansible-playbook site_rpm_repo.yml
```

### pip のリポジトリフォルダを作りインデックスを生成する(サーバ側)

- 設計として `/home/t-ando/ando/rpm/pip_repo/` の配下に作っていきます。
    - 具体手的には [site_pip_repo.yml](./ando/site_pip_repo.yml) を起動します。
    - 現状 `numpy` だけ入れています。必要なものを追加してください。

### pip のローカルリポジトリを追加する(クライアント側側)

- 配置後、pip からはこのライアント側にこのローカルアドレスを設定するようにします。
    - 具体的には以下のようなコマンドを投入します。

```shell
pip install --extra-index-url http://<NginxサーバーのIP>/repo/pip_repo/ --trusted-host <NginxサーバーのIP> <パッケージ名>
```

### ansible-galaxy のリポジトリフォルダを作りインデックスを生成する(サーバ側)

- 設計として `/home/t-ando/ando/rpm/galaxy_repo/` の配下に作っていきます。
    - 具体手的には [site_galaxy_repo.yml](./ando/site_galaxy_repo.yml) を起動します。
    - 現状 `cisco.ios` だけ入れています。必要なものを追加してください。

### TFTP を公開する

- PXE（Preboot eXecution Environment：ピー・エックス・イー）とは、ネットワーク経由でコンピュータを起動（ブート）するための規格・仕組みのことです。
- PXE を実現するためには tftp を用意する必要があります。
- 設計として `/home/t-ando/ando/rpm/` の配下が見えるようにします。
    - 具体手的には [site_tftp_start.yml](./ando/site_tftp_start.yml) を起動します。

### NTP を公開する

- NTP は `wsl` では動きません。
- NTP は windows11 では `w32time` が動いていて 123UDP を占有しているからです。
- `w32time` を NTPサーバ化する方向で考えます。
- まず管理署権限でコマンドプロンプトを開きレジストリを変更してNTPサーバー機能を有効化します。

```shell
> reg add "HKLM\SYSTEM\CurrentControlSet\Services\W32Time\TimeProviders\NtpServer" /v "Enabled" /t REG_DWORD /d 1 /f
この操作を正しく終了しました。
```

- 同様に外部のインターネット（windows.comなど）に繋がっていない完全なローカル環境の場合、自身の内蔵時計を信頼の基準（マスター）にする設定を行います。

```shell
>reg add "HKLM\SYSTEM\CurrentControlSet\Services\W32Time\Config" /v "AnnounceFlags" /t REG_DWORD /d 5 /f
この操作を正しく終了しました。
```

- `w32time` を再起動します。

```shell
> w32tm /config /update
コマンドは正しく完了しました。
>net stop w32time
Windows Time サービスを停止中です.
Windows Time サービスは正常に停止されました。
>net start w32time
Windows Time サービスを開始します.
Windows Time サービスは正常に開始されました。
>sc config w32time start=auto
[SC] ChangeServiceConfig SUCCESS
```

### DNSとDHCPをansibleで構築する

- 具体手的には [site_dhcp_start.yml](./ando/site_dhcp_start.yml) を起動します。
- 内容としては以下の通りです。
    - 自分を 192.168.10.2 にする。
    - 192.168.10.10 から 192.168.10.30 までを DHCPの範囲にする。



## 全てをまとめて実行する

- 全てをまとめて実行しましょう。
- [site.yml](./ando/site.yml) でまとめて実行できます。

```shell
ansible-playbook site.yml
```

## Windows ファイアウォールを許可する

- Windows側でファイヤーウォール許可設定をすれば完成です
