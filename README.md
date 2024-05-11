# TurtleBot3_setup
以下の内容は[TurtleBot3 e-Manual](https://emanual.robotis.com/docs/en/platform/turtlebot3/overview)
を参考に少し補足を加えたものである．

Ros2バージョンはHumble，TurtleBot3のモデルはburger，OSはUbuntu 22.04 LTSを用いる．
また，OpenCRの設定については説明しない．
# SBC Setup
TurtleBot3が積むRaspberry Piのセッティングの手順を示す．
## OS(Ubuntu 22.04LTS)の書き込み
Raspberry Pi Imagerを使用してOSをSDカードに書き込む．
1. Raspberry Pi Imagerを実行する
2. `デバイスをを選択`をクリックし，`Raspberry Pi 4`を選択
3. `CHOOSE OS`をクリックし，`Other gerneral-purpose OS`を選択
5. `Ubuntu`を選択
6. `Ubuntu Server 22.04.5 LTS (64-bit)`を選択
7. `CHOOSE STORAGE`をクリックし，microSDカードを選択する
8. `WRITE(次へ)`をクリックし，`設定を編集する`から必要な設定をする
9. 設定を保存し指示に従ってOSを書き込む
## Raspberry Piの起動とssh接続
> [!WARNING]
> 以降，時間がかかる処理が続く．よって，Raspberry Piはバッテリーではなく
> 必ず電源に接続し，電力の供給が途切れることがないようにしておく必要がある．
### ディスプレイ，キーボード，マウスを使わずにログインする場合
OSの書き込み時の設定でWiFiとSSHを有効にした場合はディスプレイ，キーボード，マウスを使わず，
起動後に別PCからSSH接続することができる．
1. ディスプレイ，キーボード，マウスを接続せずにmicroSDカードを挿入し，電源を接続してRaspberry Piを立ち上げる
2. IPアドレスを調べ，同じネットワーク内の別PCのターミナル上で`ssh {username}@{IPAddress}`もしくは`ssh {username}@{hostname}.local`と入力し接続する．
3. 初めてそのIPアドレスにアクセスするときは確認を求められるため，`yes`を入力する．
4. 以前にそのIPアドレスにSSH接続している場合はキーが一致しないという内容のエラーを吐かれるため，ホストPCの`.SSH`フォルダ内のキーを保存しているファイルから，そのIPアドレスのキーを削除し，2. 3. を行う．
5. ID，PASSでログインする
### ディスプレイ，キーボード，マウスを使って起動する場合
1. HDMIケーブルでRaspberry Piとモニターを接続．
1. マウスやキーボードをRaspberry Piに接続
1. microSDカードを挿入
1. 電源を接続してRaspberry Piを立ち上げる
1. OSの書き込み時に設定したID，PASSでログインする
> [!NOTE]
> Raspberry Piに電源を入れる前にHDMIケーブルを接続する必要がある．でなければRaspberry PiのHDMIポートが無効になる．

WifIとSSHの設定がすんでいない場合は以下の作業を行う．
#### WiFiの設定
1. 以下のコマンドでネットワーク設定ファイルを開く
```
$ sudo nano /writable/etc/netplan/50-cloud-init.yaml
```
1. エディタが開いたら，以下のように内容を編集する．ただし，`WIFI_SSID`と`WIFI_PASSWORD`を実際の SSIDとパスワードに置き換える．

```
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: yes
      dhcp6: yes
  wifis:
    wlan0
      dhcp4: yes
      dhcp6: yes
      access-points:
        WIFI_SSID:
          password: WIFI_PASSWORD
```
1. `Ctrl`+`s`でファイルを保存し，`Ctrl`+`x`で終了する．

#### SSHの設定
Ubuntu 22.04 Serverの標準的なインストールを実施したのであれば、OpenSSHサーバーはインストール済みである．
##### 有効化
サービスが起動していることを確認
```
$ systemctl status sshd.service
```
出力に`active (running)`と表示されていればOK  
インストールされていない時は次のコマンドでインストール
```
$ sudo apt update
$ sudo apt install openssh-server
```
公開鍵認証などセキュリティ面の設定は説明しない.

インストールできたらディスプレイ，キーボード，マウスを使わずにログインする場合に書いてあるように
SSH接続をする．

## 初期設定
以下のコマンドを入力して自動更新設定ファイルを編集
```
$ sudo nano /etc/apt/apt.conf.d/20auto-upgrades
```
以下のようにアップデート設定を変更
```
APT::Periodic::Update-Package-Lists "0";
APT::Periodic::Unattended-Upgrade "0";
```
`Ctrl`+`s`でファイルを保存し，`Ctrl`+`x`で終了

起動時にネットワークがない場合でも起動の遅延を防ぐために`systemd`を設定．次のコマンドを実行し，`systemd`プロセスのマスクを設定．
```
$ systemctl mask systemd-networkd-wait-online.service
```
サスペンドと休止状態を無効にする
```
$ sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```
ラズベリーパイを再起動
```
$ reboot now
```
> [!NOTE]
> 再起動したらSSHで接続できることを確かめておく．

## apt upgradeの実行
今はUbuntu 22.04LTSリリースした直後の状態のため，その後に行われたセキュリティアップデートなどが反映されていない．よって，以下のようにアップグレードを実行する
```
$ sudo apt upgrade
```
ここで様々なエラーが発生する可能性があるため，その都度対処する．

## ROS2 Humbleのインストール
[公式ROS2 Humbleインストールガイド](https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debians.html)の必要な部分のみを抜粋しシェルスクリプトにまとめた[ros2インストールスクリプト](/ros2_install.sh)．また，その実行に必要なことを補足しておく．
### ロケールの確認
`UTF-8`をサポートするロケールがあることを確認
```
$ locale  # check for UTF-8
```
なければ以下を実行
```
$ sudo apt update && sudo apt install locales
$ sudo locale-gen en_US en_US.UTF-8
$ sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
$ export LANG=en_US.UTF-8
```

### ros2のインストール
[ros2インストールスクリプト](/ros2_install.sh)を実行する．

実行内容の詳細は[公式ROS2 Humbleインストールガイド](https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debians.html)を参照

## ROSパッケージのインストールとビルド
1. [ROSパッケージインストールスクリプト](/turtlebot3_setup_01.sh)を実行し，諸々のパッケージをインストールする．
2. gitをインストールする
```
$ sudo apt install git
```
3. 以下のようにコマンドを実行する．実行内容の詳細は[TurtleBot3 e-Manual](https://emanual.robotis.com/docs/en/platform/turtlebot3/overview)参照．
```
$ mkdir -p ~/turtlebot3_ws/src && cd ~/turtlebot3_ws/src
$ git clone -b humble-devel https://github.com/ROBOTIS-GIT/turtlebot3.git
$ git clone -b ros2-devel https://github.com/ROBOTIS-GIT/ld08_driver.git
$ cd ~/turtlebot3_ws/src/turtlebot3
$ rm -r turtlebot3_cartographer turtlebot3_navigation2
$ cd ~/turtlebot3_ws/
$ echo 'source /opt/ros/humble/setup.bash' >> ~/.bashrc
$ source ~/.bashrc
$ colcon build --symlink-install --parallel-workers 1
$ echo 'source ~/turtlebot3_ws/install/setup.bash' >> ~/.bashrc
$ source ~/.bashrc
```
## OpenCRと接続するUSBポートの設定する．
```
$ sudo cp `ros2 pkg prefix turtlebot3_bringup`/share/turtlebot3_bringup/script/99-turtlebot3-cdc.rules /etc/udev/rules.d/
$ sudo udevadm control --reload-rules
$ sudo udevadm trigger
```
## `ROS_DOMAIN_ID`を設定
ROSドメインIDの設定ROS2のDDS通信では、同一ネットワーク環境下で通信するためにリモートPCとTurtleBot3の間で`ROS_DOMAIN_ID`を一致させる必要がある．次のコマンドで、TurtleBot3でSBCに`ROS_DOMAIN_ID`を割り当てる．
- TurtleBot3のデフォルトIDは30
- TurtleBot3のリモートPCとSBCのROS_DOMAIN_IDは30が推奨
```
$ echo 'export ROS_DOMAIN_ID=30 #TURTLEBOT3' >> ~/.bashrc
$ source ~/.bashrc
```
> [!WARNING]
>同じネットワーク内の他のユーザーと同じROS_DOMAIN_IDを使用してはいけない．同じネットワーク環境下にあるユーザー間で通信の競合が発生する．
## `LDS_MODEL`を設定
LDSのモデルによって`LDS_01`もしくは`LDS_02`を設定する．
```
$ echo 'export LDS_MODEL=LDS-02' >> ~/.bashrc
$ source ~/.bashrc
```


以上でラズベリーパイ側の設定はおわりである．

# エラー対策
keyringのエラーが発生することがある
```
W: GPG error: http://packages.ros.org/ros2/ubuntu jammy InRelease: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY {KEY}
```
多分，次のコマンドで解決できる
```
$ sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys F42ED6FBAB17C654
```
今，実行環境が手元に無い為詳しいことわからん
