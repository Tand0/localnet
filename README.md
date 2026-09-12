
# ansibleを使ってローカル環境の構築に必要な環境一式を作ろう

## あぶすとらくと

- インターネットから切り離されたローカル環境にサーバをおいてシステム一式を構築することはよくあります。
- そのときに困るのは rpm, apt, pip, ansible-galaxy などのリポジトリです。
- その他 TFTP, DNS, DHCP, NTP, HTTP なども環境として用意が必要でしょう。
- そんなときには ansible!  ということで、環境一式をwslを使って構築し、PC上で動かせるようにします。
- イメージとして以下の通りです。

![イメージ](./img/002pc.jpg)

## 基本パラメータ

- 基本的なパラメータとしては以下にしてみましょう。
    - `/home/user/ando` -- ベースのフォルダ
    - `/home/user/ando/repo` -- 各ファイルを置くディレクトリ
    - 192.168.10.3 -- インストールを実施するPC(というか ubuntu)
    - 192.168.10.0/24 -- ローカル環境
    - 192.168.10.10-200 -- インストール先の PC

## もくじ

- [ansibleを使ってローカル環境の構築に必要な環境一式を作ろう](#ansibleを使ってローカル環境の構築に必要な環境一式を作ろう)
    - [あぶすとらくと](#あぶすとらくと)
    - [基本パラメータ](#基本パラメータ)
    - [もくじ](#もくじ)
    - [初期環境構築](#初期環境構築)
        - [ansible-vault によるパスワードの設定](#ansible-vault-によるパスワードの設定)
        - [評価用に cisco のルータのシミュレータを用意する](#評価用に-cisco-のルータのシミュレータを用意する)
    - [各種リポジトリの作成](#各種リポジトリの作成)
        - [各種リポジトリフォルダを nginx で公開する](#各種リポジトリフォルダを-nginx-で公開する)
        - [PXE 用に TFTP を公開する](#pxe-用に-tftp-を公開する)
        - [NTP を公開する](#ntp-を公開する)
        - [DNSとDHCPをansibleで構築する](#dnsとdhcpをansibleで構築する)
        - [rpm のリポジトリフォルダを作りインデックスを生成する(サーバ側)](#rpm-のリポジトリフォルダを作りインデックスを生成するサーバ側)
        - [rpm のローカルリポジトリを追加する(クライアント側)](#rpm-のローカルリポジトリを追加するクライアント側)
        - [pip のリポジトリフォルダを作りインデックスを生成する(サーバ側)](#pip-のリポジトリフォルダを作りインデックスを生成するサーバ側)
        - [pip のローカルリポジトリを追加する(クライアント側側)](#pip-のローカルリポジトリを追加するクライアント側側)
        - [ansible-galaxy のリポジトリフォルダを作りインデックスを生成する(サーバ側)](#ansible-galaxy-のリポジトリフォルダを作りインデックスを生成するサーバ側)
    - [全てをまとめて実行する](#全てをまとめて実行する)

## 初期環境構築

- まずはインストール時の基本設定です
    - ubuntu の PC の環境はこんな感じです
    - システム設定 ⇒ 日本語
    - キーボードレイアウト ⇒ Japanese/Japanese
    - 地域 ⇒ Tokyo
    - あなたの名前 ⇒ user
    - コンピュータの名前 ⇒ myserver
    - ユーザの名前 ⇒ user
    - パスワード ⇒ abcd1234
    - パスワードの確認 ⇒ abcd1234
    - ログイン時にパスワードを要求する
- settings で IP を設定します
    - Network
        - wired
            - Identify
                - Name: Wired connection 1
                - MAC Adress: 00:15:5D:F0:C0:0E (eth0)
                - Cloned Address: Permanent
                - MTU: automatic
            - IPv4
                - IPv4 Method: Manual
                - Address: 192.168.10.3 255.255.255.0 192.168.10.2
                - DNS: Automatic
            - IPv4
                - IPv6 Method: Disable
            - Security:
                - 802.1x Security: off
- ターミナルを開いて状況確認します

```shell
$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:15:5d:f0:c2:10 brd ff:ff:ff:ff:ff:ff
    inet 172.25.170.117/20 brd 172.25.175.255 scope global dynamic noprefixroute eth0
       valid_lft 86111sec preferred_lft 86111sec
    inet6 fe80::839d:adb8:16f6:2ff7/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:15:5d:f0:c2:11 brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.3/24 brd 192.168.10.255 scope global noprefixroute eth1
       valid_lft forever preferred_lft forever
$ cat /etc/os-release
PRETTY_NAME="Ubuntu 22.04 LTS"
NAME="Ubuntu"
VERSION_ID="22.04"
VERSION="22.04 LTS (Jammy Jellyfish)"
VERSION_CODENAME=jammy
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=jammy
$ pwd
/home/user
```

- とりあえず `/home/t-ando/ando` を作ってそこで作業します

```shell
$ sudo apt update
$ sudo apt upgrade -y
$ sudo apt install openssh-server
$ sudo apt install curl
$ sudo apt install tftp
$ python3 --version
Python 3.10.4
$ sudo apt install -y python3.10-venv
$ cd 
$ mkdir ando
$ cd ando
$ pwd
/home/user/ando
$ python3 -m venv .env
$ source .env/bin/activate
$ pip install ansible
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
p_pass: abcd1234
p_pass_sudo: abcd1234
p_pass_enable: abcd1234
p_pass_local: abcd1234
p_pass_local_sudo: abcd1234
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
p_pass: abcd1234
p_pass_sudo: abcd1234
p_pass_enable: abcd1234
p_pass_local: abcd1234
p_pass_local_sudo: abcd1234
```

- 実行時は `--ask-vault-pass` を指定することで１回のパスワード入力でまとめてパスワードが投入できます。

```shell
ansible-playbook site.yml --ask-vault-pass
```

- [ansible.cfg](./ando/ansible.cfg) で以下のように書けば `--ask-vault-pass` も不要になります。

```ini
[defaults]
inventory = hosts.yml
ask_vault_pass = True
host_key_checking = False
```

```shell
ansible-playbook site.yml
```

- `hosts.yml` も設定しましょう

```yml
all:
  hosts:
    rhel_ssh:
      ansible_host: 192.168.10.5
      ansible_port: 22
      ansible_user: user
      ansible_password: "{{p_pass}}"
      ansible_become_password: "{{p_pass_local_sudo}}"
      ansible_ssh_common_args: '-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null'
    localhost:
      ansible_connection: local
      ansible_host: localhost
      ansible_port: 22
      ansible_python_interpreter: "/home/user/ando/.env/bin/python"
      ansible_password: "{{p_pass_local}}"
      ansible_become_password: "{{p_pass_local_sudo}}"
```

### 評価用に cisco のルータのシミュレータを用意する

- オプションです。
- cisco のルータなどを ansible-playbook で叩きたいが手元にない場合、シミュレータで動作確認します。
- そのために GNS3 を使います。
- [https://www.gns3.com/](https://www.gns3.com/) を参照してください。

## 各種リポジトリの作成

### 各種リポジトリフォルダを nginx で公開する

- 起動は [site_nginx_start.yml](./ando/site_nginx_start.yml) で実施します。
- 停止は [site_nginx_stop.yml](./ando/site_nginx_stop.yml) で実施します。
- どちらも `ansible-playbook site_XXXX.yml` で実行できます。
- ここでは `ando/repo` フォルダ配下を公開しています
- `https://192.168.10.3/` や `https://192.168.10.3/tftp_repo` が外部からアクセスできることを確認してください

### PXE 用に TFTP を公開する

- PXE（Preboot eXecution Environment：ピー・エックス・イー）とは、
  ネットワーク経由でコンピュータを起動（ブート）するための規格・仕組みのことです。
- PXE を実現するためには tftp を用意する必要があります。
- 設計として `/home/user/ando/repo/tftp_repo/` の配下が見えるようにします。
- 起動は [site_tftp_start.yml](./ando/site_tftp_start.yml) で実施します。
- 停止は [site_tftp_stop.yml](./ando/site_tftp_stop.yml) で実施します。
- どちらも `ansible-playbook site_XXXX.yml` で実行できます。
- `tftp 192.168.10.3` が外部からアクセスできることを確認してください。

### NTP を公開する

- NTPは時刻の同期に使用します。ローカル環境で動かすには工夫が必要です。
- 設定として `/etc/chrony/chrony.conf` に `local` の指定が必要です。
- 起動は [site_ntp_start.yml](./ando/site_ntp_start.yml) で実施します。
- 停止は [site_ntp_stop.yml](./ando/site_ntp_stop.yml) で実施します。
- どちらも `ansible-playbook site_XXXX.yml` で実行できます。
- `chronyc tracking` で動作を確認してください。

### DNSとDHCPをansibleで構築する

- DNSはIPとPXEのファイルを指定します。
- 起動は [site_dhcp_start.yml](./ando/site_dhcp_start.yml) で実施します。
- 停止は [site_dhcp_stop.yml](./ando/site_dhcp_stop.yml) で実施します。
- どちらも `ansible-playbook site_XXXX.yml` で実行できます。
- 内容としては以下の通りです。
    - 接続先の interface は `ip a` で確認する。
    - 自分を 192.168.10.3 にする。
    - 192.168.10.10 から 192.168.10.30 までを DHCPの範囲にする。
    - tftp の接続先を 192.168.10.3 にする

```yml
    wsl_dummy_interface: "eth1"  # please check ip a
    wsl_dummy_ip: "192.168.10.3"

    dhcp_subnet: "192.168.10.0"
    dhcp_netmask: "255.255.255.0"
    dhcp_range_start: "192.168.10.5"
    dhcp_range_end: "192.168.10.100"
    dhcp_router: "192.168.10.1"

    tftp_server_ip: "192.168.10.3" 
```

- 動作を以下のコマンドで確認してください。

```shell
sudo systemctl status bind9
sudo systemctl status isc-dhcp-server
```

- tftp 用に ubuntu のファイルをダウンロードしてきます。

```shell
mkdir -p /tmp/pxe-extract && cd /tmp/pxe-extract
apt-get download shim-signed grub-efi-amd64-signed
dpkg-deb -x shim-signed_*.deb shim/
dpkg-deb -x grub-efi-amd64-signed_*.deb grub/
sudo cp shim/usr/lib/shim/shimx64.efi.signed.latest /home/user/ando/repo/tftp_repo/bootx64.efi
sudo cp grub/usr/lib/grub/x86_64-efi-signed/grubnetx64.efi.signed /home/user/ando/repo/tftp_repo/bootx64.efi
```

### rpm のリポジトリフォルダを作りインデックスを生成する(サーバ側)

- rpm は RHELでインストールを行うときに必要となるファイルです。
- このリポジトリを作ってローカルからアクセスできるようにします。
- 起動は [site_rpm_repo.yml](./ando/site_rpm_repo.yml) で実施します。
- `ansible-playbook site_XXXX.yml` で実行できます。
- 設計として `/home/t-ando/ando/rpm/rpm_repo/` の配下に作成します。

### rpm のローカルリポジトリを追加する(クライアント側)

- rpm は RHELでインストールを行うときに必要となるファイルです。
- 配置後、クライアント側にこのローカルアドレスを設定するようにします。
- 起動は [site_rhel_rpm.yml](./ando/site_rhel_rpm.yml) で実施します。
- `ansible-playbook site_XXXX.yml` で実行できます。
    - `local_repo_url` は適宜変更が必要です。

### pip のリポジトリフォルダを作りインデックスを生成する(サーバ側)

- pip は python3でパッケージをインストールを行うときに必要となるファイルです。
- 起動は [site_pip_repo.yml](./ando/site_pip_repo.yml) で実施します。
- `ansible-playbook site_XXXX.yml` で実行できます。
- 設計として `/home/t-ando/ando/rpm/pip_repo/` の配下に作っていきます。
    - 現状 `numpy` だけ入れています。必要なものを追加してください。

### pip のローカルリポジトリを追加する(クライアント側側)

- 配置後、pip からはこのライアント側にこのローカルアドレスを設定するようにします。
    - 具体的には以下のようなコマンドを投入します。

```shell
pip install --extra-index-url http://【NginxサーバーのIP】/repo/pip_repo/ --trusted-host 【NginxサーバーのIP】 【パッケージ名】
```

### ansible-galaxy のリポジトリフォルダを作りインデックスを生成する(サーバ側)

- ansible-galaxy は ansible で cisco.ios のようなコレクションをインストールするときに必要となる環境です。
- 起動は [ansible-galaxy.yml](./ando/ansible-galaxy.yml) で実施します。
- `ansible-playbook site_XXXX.yml` で実行できます。
- 設計として `/home/t-ando/ando/rpm/galaxy_repo/` の配下に作っていきます。
    - 現状 `cisco.ios` だけ入れています。必要なものを追加してください。

## 全てをまとめて実行する

- 全てをまとめて実行しましょう。
- [site.yml](./ando/site.yml) でまとめて実行できます。

```shell
ansible-playbook site.yml
```
