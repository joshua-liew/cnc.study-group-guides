# FreeRADIUS について


## Installation (ソースからビルド)

以下のサイトからパッケージをダウンロードすることができます：
- https://github.com/FreeRADIUS/freeradius-server/releases
- https://www.freeradius.org/releases/

1. まず、依存関係 (`libtalloc`, `libkqueue`, `openssl`, など) のインストールを行います：
    ```
    sudo apt-get update
    apt-get install libtalloc-dev libkqueue-dev libssl-dev
    ```

2. ソースパッケージをダウンロードします。このチュートリアルではバージョン `3.2.7` を使っていますが、適宜に **最新の安定版リリース** をダウンロードしてください。
    ```
    # /tmp ディレクトリ にパッケージをダウンロードする
    cd /tmp/
    wget https://github.com/FreeRADIUS/freeradius-server/releases/download/release_3_2_7/freeradius-server-3.2.7.tar.gz
    tar -xvf freeradius-server-3.2.7.tar.gz
    ```

3. `--prefix` オプションでインストール先を指定できます。このチュートリアルでは `/home/user/freeradius` ディレクトリに FreeRADIUSサーバ をインストールします。
    ```
    cd freeradius-server-3.2.7/
    ./configure --prefix=$HOME/freeradius
    make
    sudo make install
    ``` 
    > ビルドでつまずいてしまう場合、`stdout` の出力結果を読んでください。依存関係でビルドできないケースが多いので、何が必要かは必ず `stdout` に記載されています。

4. `freeradius` ディレクトリの中身が以下のようになっていることを確認してください。
    ```
    cd ~/freeradius/
    tree -d -L 1
    .
    ├── bin
    ├── etc
    ├── include
    ├── lib
    ├── sbin
    ├── share
    └── var
    ```
    > 各ディレクトリの説明を簡単に述べます。
    > - `bin`：FreeRADIUSサーバの提供する機能を実装するためのバイナリ（実行ファイル）が置いてあるディレクトリ
    > - `etc`：FreeRADIUSサーバの設定ディレクトリ
    > - `include`：FreeRADIUSは **C言語** で書かれているので、ヘッダーファイル（定数や宣言などを書くためのファイル）が置いてあるディレクトリ
    > - `lib`：FreeRADIUSは **C言語** で書かれているので、リンクするライブラリーが置いてあるディレクトリ
    > - `sbin`：FreeRADIUSサーバ自体（サーバを動作させるため）のバイナリが置いてあるディレクトリ
    > - `share`：ドキュメントやRADIUSプロトコルを実装するための追加情報や仕様などが置いてあるディレクトリ
    > - `var`：`log` が置いてあるディレクトリ

5. バイナリをパスに追加します。`./configure` で指定したインストール先に従ってパスを指定してください。`.bashrc` や `.profile` に追加すると良いでしょう。
    ```
    # export PATH="$PATH:<バイナリディレクトリへのパス>"
    export PATH="$PATH:$HOME/freeradius/sbin"
    export PATH="$PATH:$HOME/freeradius/bin"
    ```
    > `.bashrc` や `.profile` で設定した場合、設定後に必ず設定を適用してください。
    > ```
    > source .bashrc
    > ```
    
    `systemd` でサーバを管理することも可能ですが、`systemctl` のデーモンに必要な情報を渡さなければいけません。

6. インストールの確認を行います。
    ```
    # helpメニュを表示する
    radiusd -h
    Usage: radiusd [options]
    Options:
    ...
      -X            Turn on full debugging (similar to -tfxxl stdout).
    ...

    # デバッグモードでサーバを起動させる
    radiusd -X
    ```

    正常に動作している場合、以下が `stdout` に出力されるはずです。
    ```
    FreeRADIUS Version 3.2.7
    Copyright (C) 1999-2023 The FreeRADIUS server project and contributor    
    
    ...
    Starting - reading configuration files ...
    including dictionary file /home/joshua/freeradius/share/freeradius/dictionary
    including dictionary file /home/joshua/freeradius/share/freeradius/dictionary.dhcp    
    
    ...
    Listening on auth address 127.0.0.1 port 18120 bound to server inner-tunnel
    Listening on auth address * port 1812 bound to server default
    Listening on acct address * port 1813 bound to server default
    Listening on auth address :: port 1812 bound to server default
    Listening on acct address :: port 1813 bound to server default
    Listening on proxy address * port 54819
    Listening on proxy address :: port 60719
    Ready to process requests
    ```
    > エラーが出た場合、`stdout` を読んで適宜に対応してください。


## Port (ポート)

RADIUSプロトコルに正式に割り当てられた **UDPポート番号** は **`1812`** です。
- `1812`：RADIUSの **認証** の使用しているポート
- `1813`：RADIUSの **アカウンティング** の使用しているポート

サーバのマシンでファイアウォールがかかっている場合、必要なポートを必ず開放してください。また、ファイアウォールルーターの配下にRADIUSサーバを設置した場合、必要に応じて RADIUSパケット（認証パケットとアカウンティングパケット）を通してください。

> 参考：RFC2865に正式に定義されています。
> - 英語版：[https://datatracker.ietf.org/doc/html/rfc2865](https://datatracker.ietf.org/doc/html/rfc2865)
> - 日本語版：s[https://tex2e.github.io/rfc-translater/html/rfc2865.html](https://tex2e.github.io/rfc-translater/html/rfc2865.html)


## RADIUSプロトコルの概要
> 参考書：[RADIUS ― ユーザ認証セキュリティプロトコル（Jonathan Hassell 著、株式会社アクセンス・テクノロジー 訳）、発行年月日：2003年12月](https://www.oreilly.co.jp/books/4873111471/)


### AAAの概要

RADIUSプロトコルが基盤としているフレームワークは、一般に「**AAAモデル**」と呼ばれており、 **認証(Authentication), 承認(Authorization), アカウンティング(Accounting)** の3つで構成されています。RADIUSプロトコルの多くの実装において実際に AAAモデル が実現されています。

AAAのモデルの役割は、全てのトランザクションの開始から終了までを管理、そして報告することです。例えば、次のような質問が、AAAモデル の各機能をうまく表現しています。
- 相手はだれなのか？
    - **`認証(Authentication)`** とは、相手の示した身分を確認する処理です。ログインIDとパスワードを用いた認証方式（`EAP-PEAP`, etc）や 公開鍵基盤(PKI)の公開鍵証明書(デジタル証明書)を用いた認証方式（`EAP-TLS`, etc） で認証を行います。
- どんなサービスをユーザに与えるよう承認されているのか？
    - **`承認(Authorization)`** とは、ユーザにどのようなことを許すのかを決定するためのポリシーを設定（そして、適用）する処理です。例えば、学生用ネットワーク、ビジター用ネットワークの3つを用意しておき、ネットワークに接続するユーザの属性に応じて接続先のネットワークを切り替えることを可能にする「`ダイナミック認証VLAN`」が承認の例の一つです。[\[1\]: NIIによるダイナミック認証VLANの設定例](https://meatwiki.nii.ac.jp/confluence/pages/viewpage.action?pageId=25560781)
- ユーザがサービスを利用している最中にどんなことを行ったのか？
    - **`アカウンティング`** とは、ユーザがサービスを利用した事実と、その内容を計測し、記録する処理です。例えば、サービスを提供した時間（秒数）、送受信したデータ量などが含まれます。[\[2\]: FreeRADIUSのアカウンティング機能の実装のためのDB schema](https://github.com/FreeRADIUS/freeradius-server/tree/master/raddb/mods-config/sql/main)

### FreeRADIUSにおけるAAAの実装

FreeRADIUSでは、AAAモデルが実現されています。設定ディレクトリが `etc/raddb` ディレクトリになっています。
```
cd freeradius/etc/raddb/

# stdoutに設定情報を、コメントを除いて出力する
sed -E '/^[[:blank:]]*(#|$)/d; s/#.*//' sites-enabled/default
```

設定の中に、`authorize`, `authenticate`, `accounting` に該当するセクションがあることが分かります。
```
server default {
listen {...}
authorize {
        filter_username
        preprocess
        chap
        mschap
        digest
        suffix
        eap {
                ok = return
        }
        files
        -sql
        -ldap
        expiration
        logintime
        pap
        Autz-Type New-TLS-Connection {
                  ok
        }
}
authenticate {
        Auth-Type PAP {
                pap
        }
        Auth-Type CHAP {
                chap
        }
        Auth-Type MS-CHAP {
                mschap
        }
        mschap
        digest
        eap
}
preacct {...}
accounting {
        detail
        -sql
        exec
        attr_filter.accounting_response
}
session {
}
post-auth {...}
...
}
```

### RADIUSパケット形式について

参考：[RFC2865: Packet Format](https://tex2e.github.io/rfc-translater/html/rfc2865.html#3--Packet-Format)

RADIUSは、SMTPやHTTPのようなテキストベースなプロトコルと違って、**バイナリ転送プロトコル（バイナリデータの送受信を行なう）** になっています。[\[3\]](https://www.networkradius.com/doc/current/concepts/dictionary/introduction.html) バイナリでデータを転送しているため、人間に読める状態にするのに `dictionary` というものを用いて、バイナリデータを特定の文字列にマッピングします。FreeRADIUSでは、マッピングの定義はすべて `share/freeradius` ディレクトリにあります。
```
cd freeradius/
ls share/freeradius/
... dictionary.rfc2865  #RFC2865を実現するためのマッピング

cat share/freeradius/dictionary.rfc2865
ATTRIBUTE       User-Name                               1       string
ATTRIBUTE       User-Password                           2       string encrypt=1
ATTRIBUTE       CHAP-Password                           3       octets
ATTRIBUTE       NAS-IP-Address                          4       ipaddr
ATTRIBUTE       NAS-Port                                5       integer
...
```

> 重要：バイナリ転送プロトコルなので、ベンダーによってマッピングが異なる可能性があることに注意しましょう。

RADIUSデータ形式の概要を以下に示します。フィールドは左から右に送信されます。
```
 0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |     Code      |  Identifier   |            Length             |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                                                               |
   |                         Authenticator                         |
   |                                                               |
   |                                                               |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  Attributes(属性) ...
   +-+-+-+-+-+-+-+-+-+-+-+-+-
```

> `radiusd -X` の出力からも、dictionaryファイル（マッピング）が読み込まれていることが分かります。
> ```
> Starting - reading configuration files ...
> including dictionary file /home/joshua/freeradius/share/freeradius/dictionary
> including dictionary file /home/joshua/freeradius/share/freeradius/dictionary.dhcp
> including dictionary file /home/joshua/freeradius/share/freeradius/dictionary.vqp
> ```


## FreeRADIUSの基本操作
> `etc/raddb` ディレクトリにおいて、主な設定を行います。`apache2` に似たようなアーキテクチャになっています。特に `mods-enabled` や `sites-enabled` のところに見覚えがあるでしょう。
```
cd freeradius/etc/raddb/
tree -L 1
.
├── README.rst
├── certs/
├── clients.conf        #重要：どこからRADIUS認証リクエストを受け取るか設定
├── dictionary
├── experimental.conf
├── hints -> ./mods-config/preprocess/hints
├── huntgroups -> ./mods-config/preprocess/huntgroups

├── mods-available/     #重要：使用できるモジュール
├── mods-config/        #重要：モジュールの使用する設定
├── mods-enabled/       #重要：有効化されたモジュール

├── panic.gdb

├── policy.d/           #重要：承認に使用されるポリシー設定

├── proxy.conf          #重要：realmやproxyの設定
├── radiusd.conf        #重要：FreeRADIUSサーバのconfig(設定)ファイル

├── sites-available/    #重要：使用できるバーチャルRADIUSサーバ
├── sites-enabled/      #重要：有効化されたバーチャルRADIUSサーバ

├── templates.conf
├── trigger.conf
└── users -> ./mods-config/files/authorize
```

### `radiusd.conf`：FreeRADIUSサーバの設定ファイル
```
cd freeradius/etc/raddb/
sed -E '/^[[:blank:]]*(#|$)/d; s/#.*//' radiusd.conf
----------------------------------------------------
# 変数の定義
prefix = /home/joshua/freeradius
exec_prefix = ${prefix}
sysconfdir = ${prefix}/etc
localstatedir = ${prefix}/var
...
# ログの設定
log {
        destination = files
        colourise = yes
        file = ${logdir}/radius.log
        syslog_facility = daemon
        stripped_names = no
        auth = no
        auth_badpass = no
        auth_goodpass = no
        msg_denied = "You are already logged in - access denied"
}
...
$INCLUDE proxy.conf         # proxyの設定を読み込む
$INCLUDE clients.conf       # clientsの設定を読み込む
...
# 有効化されたモジュールのみ読み込む
modules {
        $INCLUDE mods-enabled/ 
}
instantiate {
}
policy {
        $INCLUDE policy.d/
}
# 有効化されたバーチャルRADIUSサーバのみ読み込む
$INCLUDE sites-enabled/
```

### `clients.conf`：どこからRADIUSパケットを受け取るかの設定ファイル
```
cd freeradius/etc/raddb/
sed -E '/^[[:blank:]]*(#|$)/d; s/#.*//' clients.conf
----------------------------------------------------
# eduroam用の設定例
client localhost {
        ipaddr = 127.0.0.1
        proto = *
        secret = [パスワード]
        require_message_authenticator = no
        nas_type         = other
        limit {
                max_connections = 16
                lifetime = 0
                idle_timeout = 30
        }
}
client rlab_network {
        ipaddr          = [IPv4アドレス]
        ...
        virtual_server  = eduroam
}
```

### `proxy.conf`：`realm` や `proxy` の設定ファイル
```
cd freeradius/etc/raddb/
sed -E '/^[[:blank:]]*(#|$)/d; s/#.*//' proxy.conf
--------------------------------------------------
# eduroam用の設定例
proxy server {
        default_fallback = no
}
# 自機関宛の認証リクエストを、このサーバで処理する
realm [自機関のドメイン] {
        authhost        = LOCAL
        accthost        = LOCAL
}
...
# 他機関宛の認証リクエストを、国レベルのRADIUSサーバにプロキシする
# 国レベルのRADIUSサーバは、正しい機関に認証リクエストを転送する
home_server [国レベルのRADIUSサーバ] {
        type            = auth+acct
        ipaddr          = [IPv4アドレス]
        port            = 1812
        ...
}
...
realm NULL {
        virtual_server  = nonexistent
}
realm DEFAULT {
        pool            = jp-top
        nostrip
}
```