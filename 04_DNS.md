DNSサーバの環境構築について
========================================

# 概要
DNSサーバの構築方法を示す。  
DNSサーバではOS及びBIND（DNS用ソフトウェア）のインストール等を研修前に完了させておき、  
研修内で設定ファイルおよびゾーンファイルの一部修正を行う想定になっている。

# 構築手順
DNSサーバの構築例を記載する。本説明ではAlmaLinux（RHELクローン）上に環境を構築する手順を説明するが、  
他のOSであってもそれぞれのOSに合った手順でDNSサーバを導入することができる。

## 構築環境前環境の構築手順
### bindのインストール
1. AlmaLinuxの端末を開き、rootアカウントに切り替える
```
su
```
2. BINDをインストールする。１回確認[y/N]が求められるため、yを選択する。
```
dnf install bind bind-utils
```
3. インストールされたBINDのバージョンを確認する。
```
named -v
```
### bindの設定
以下、研修用の設定を入れる。
1. named.confを設定する
```
vi /etc/named.conf
```
以下、の内容に書き換える。  

named.conf
```
options {
        listen-on port XX { XXX; };     //DNSのIPアドレス
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        secroots-file   "/var/named/data/named.secroots";
        recursing-file  "/var/named/data/named.recursing";
        allow-query { any; };

        recursion yes;

        dnssec-validation yes;

        managed-keys-directory "/var/named/dynamic";
        geoip-directory "/usr/share/GeoIP";

        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";

        include "/etc/crypto-policies/back-ends/bind.config";
};

logging {
        channel default_debug {
        file "data/named.run";
        severity dynamic;
        };
};

zone "salgroup.local" IN {
        type forward;
        forwarders { 192.168.2.151; }; // 正しいADサーバのIPアドレスを指定
};

#zone "salgroup.com" IN {
#       type master;
#       file "/var/named/salgroup.com.zone";
#};
```

2. zoneファイルを作成する。

```
vi /var/named/salgroup.com.zone
```

以下の内容を記載する。  

salgroup.com.zone
```
$TTL 86400
@   IN  SOA     ns.salgroup.com. root.salgroup.com. (
                        2024051001 ; Serial
                        3600       ; Refresh
                        1800       ; Retry
                        604800     ; Expire
                        86400      ; Minimum TTL
                )

@       IN      NS      ns.salgroup.com.
ns     IN      A       XXX
www     IN      A       XXX
```

以上の手順により、構築演習前環境の構築が完了する。

## 構築演習後環境の構築手順

### confファイルの設定
1. confファイルを編集する
```
vi /var/named/salgroup.com.zone
```

1. DNSサーバの受信設定

    - 修正前
        ```
        listen-on port XX { XXX; };	//DNSのIPアドレス
        ```
    - 修正後
        ```
        listen-on port 53 { 192.168.3.111; };	//DNSのIPアドレス
        ```
    - 上記の設定は、DNSサーバの53番ポートで受信した通信をBINDで処理することを意味している<br><br>

1. BINDを権威DNSとして設定する

    - 修正前
        ```
        #zone "salgroup.com" IN {
        #	type master;
        #	file "/var/named/salgroup.com.zone";
        #};
        ```
    - 修正後(#を消して、コメントアウトを解除)
        ```
        zone "salgroup.com" IN {
            type master;
            file "/var/named/salgroup.com.zone";
        };
        ```
   

## ゾーンファイルの設定

1. ゾーンファイルを編集する
```
vi /var/named/salgroup.com.zone
```


- DNSサーバとWebサーバのドメイン名をIPに変換するルールを定義
    - 修正前
        ```
        $TTL    86400
        @   IN  SOA     ns.salgroup.com. root.salgroup.com.{
                        2024051001  ;Serial
                        3600        ;Refresh
                        1800        ;Retry
                        604800      ;Expire
                        86400       ;Minimum TTL
        }
        @   IN  NS      ns.salgroup.com.
        ns  IN  A       XXX
        www IN  A       XXX
        ```
    - 修正後(nsおよびwwwのIPアドレスを修正)
        ```
        $TTL    86400
        @   IN  SOA     ns.salgroup.com. root.salgroup.com.{
                        2024051001  ;Serial
                        3600        ;Refresh
                        1800        ;Retry
                        604800      ;Expire
                        86400       ;Minimum TTL
        }
        @   IN  NS      ns.salgroup.com.
        ns  IN  A       192.168.3.111
        www IN  A       192.168.3.131
        ```


## BINDの再起動

1. 設定変更を反映させるために、BINDを再起動する
    ```c
    systemctl restart named.service
    ```
    <br><br>

1. DNSが機能しているか確認するために、BINDのステータスを確認する
    ```c
    systemctl status named.service
    ```
    - active (running) になっていればOK<br>
    ※ステータス確認を終えて、再度コマンドを入力するときはqを入力する

以上の手順により、構築演習後環境の構築が完了する。