
# 🧭 WSL における Guix 1.5.0 導入手順

目的：WSL 上に Guix のパッケージ管理機能を備えた環境（Guix 1.5.0）を構築。
前提：WSL2（Ubuntu）、root 権限

## 📌 この手順が対象とするもの

- **Guix System のインストール** ❌ ではない
- **Guix を Debian/Ubuntu と同じパッケージマネージャとして使う** ✅
- **Guix System のような完全な systemd 統合は想定していない手順** ✅
- デーモンは guix pull 実行後、systemd により WSL 起動時に自動起動されます ✅

## 1. 旧 Guix が存在する場合は削除（クリーンスタート）が必要

既存の Guix が残っていると、バージョンの混在や不具合の原因になるため、完全に削除します：

```bash
rm -rf /gnu
rm -rf /var/guix
```

WSL では systemd がないため、Guix System のような複雑な依存は存在しない。
この 2 つを消せば Guix の世界は完全にリセットされる。

## 2. Guix 1.5.0 バイナリを取得

```bash
wget https://ftp.gnu.org/gnu/guix/guix-binary-1.5.0.x86_64-linux.tar.xz
```

## 3. Guix 1.5.0 バイナリを展開

```bash
tar -xf guix-binary-1.5.0.x86_64-linux.tar.xz
mv gnu /gnu
mv var/guix /var/
```

Guix の世界（/gnu/store と /var/guix）が 1.5.0 の状態で再構築される。

## 4. デーモンを起動（初期段階では手動起動）

展開されたディレクトリを確認：

```bash
ls /gnu/store/ | grep guix
```

このコマンドで `xxxxxxx-guix-1.5.0` 形式のディレクトリが見つかります。以下でデーモンを起動：

```bash
/gnu/store/*-guix-1.5.0/bin/guix-daemon --build-users-group=guixbuild
```

この段階では systemd service が無いため、デーモンは手動で起動する必要があります。guix pull 実行後、systemd により自動管理されるようになります。

### 4.1 デーモンの起動（1.5.0 バイナリ段階）

**デーモンは起動後、起動させ続ける必要があります**

この段階ではまだ systemd service が無いため、以下の選択肢から選ぶ：

**方法1: 起動させ続ける（推奨）**

デーモンを一度起動したら、起動させ続けます：
```bash
/gnu/store/*-guix-1.5.0/bin/guix-daemon --build-users-group=guixbuild
```

root セッションで上記コマンドを実行するだけで、WSL インスタンス全体で daemon が利用可能になります。複数のターミナルウィンドウを開いても、すべて同じ WSL インスタンスを共有するため、一度起動すれば十分です。

**方法2: 毎回手動で起動**

Guix を使う度に以下を実行：
```bash
sudo /gnu/store/*-guix-1.5.0/bin/guix-daemon --build-users-group=guixbuild &
```

## 5. substitute（代替バイナリ）鍵の登録

⚠️ **重要**: この時点では `guix` コマンドはまだ PATH に登録されていないため、フルパスで実行する必要があります。

substitute（Guix 公式とミラーからのビルド済みバイナリ）を使用できるように、公開鍵を登録します。

ci.guix.gnu.org（公式ビルドファーム）

```bash
/gnu/store/*-guix-1.5.0/bin/guix archive --authorize \
  < /gnu/store/*-guix-1.5.0/share/guix/ci.guix.gnu.org.pub
```

bordeaux.guix.gnu.org（ミラー）

```bash
/gnu/store/*-guix-1.5.0/bin/guix archive --authorize <<EOF
(public-key
 (ecc
  (curve Ed25519)
  (q #7D602902D3A2DBB83F8A0FB98602A754C5493B0B778C8D1DD4E0F41DE14DE34F#)
  )
 )
EOF
```

これで substitute が 100% 利用可能になります。以降のコマンド実行が大幅に高速化されます。

## 6. 最新の Guix を取得（guix pull）

```bash
/gnu/store/*-guix-1.5.0/bin/guix pull
```

WSL でも substitute が効くため高速に完了する。

### 6.1 guix pull 実行時の systemd 自動設定

`guix pull` 実行時、guix が systemd service ファイル（`/etc/systemd/system/guix-daemon.service`）を自動生成します。その後、systemd により daemon が以下のように自動管理されます：

- WSL 起動時に自動起動
- WSL シャットダウン時に自動停止
- 予期しないクラッシュ時の自動再起動

自動設定されたかどうかを確認：

```bash
systemctl is-enabled guix-daemon
```

出力が `enabled` の場合、daemon は WSL 起動時に自動起動が有効になっています。この場合、セクション 4.1 の手動起動は不要です。以降は WSL 起動時に自動的に daemon が起動されます。

## 7. 環境変数の設定（Guix を PATH に通す）

### 7.1 一回限りの設定

以下を実行すると、その時だけ `guix` コマンドが使用可能になります：

```bash
GUIX_PROFILE="/root/.config/guix/current"
. "$GUIX_PROFILE/etc/profile"
unset GUIX_PROFILE
```

### 7.2 永続的な設定（推奨：シェル起動時に自動適用）

毎回セクション7.1を実行するのではなく、`/root/.bashrc` に以下を追加して永続化します：

```bash
{
  echo ''
  echo '# Guix environment'
  echo 'export GUIX_PROFILE="/root/.config/guix/current"'
  echo '. "$GUIX_PROFILE/etc/profile"'
} >> /root/.bashrc
```

その後、設定を反映させるため：

```bash
source /root/.bashrc
```

以降、新しいシェル起動時に自動的に `guix` コマンドが使用可能になります。

### 7.3 一般ユーザーでの利用設定

各ユーザーのシェル起動スクリプト（`~/.bashrc` など）に以下を追加：
```bash
{
  echo ''
  echo '# Guix environment'
  echo 'export GUIX_PROFILE="$HOME/.config/guix/current"'
  echo '. "$GUIX_PROFILE/etc/profile"'
} >> ~/.bashrc
```

その後、設定を反映させるため：

```bash
source ~/.bashrc
```

以降、新しいシェル起動時に自動的に `guix` コマンドが使用可能になります。

**重要な注意**：
- PATH 設定だけで `guix --version`、`guix package -A` などの読み取り操作は使用可能
- ただし、`guix install` などの操作にはデーモンが必要です
- デーモンが起動していない場合、"Connection refused" エラーになります
- デーモンは root で起動・保持される必要があります（セクション 4.1 参照）

## 8. 動作確認

バージョン確認

```bash
guix --version
```

チャンネル確認

```bash
guix describe
```

substitute の状態確認

```bash
guix weather hello
```

## 9. これで完成：WSL 上の Guix が完全に正常化

1.4.0 の残骸なし

1.5.0 の世界をクリーンに展開

デーモンは 1.5.0 → pull 後は最新

substitute 100% 利用可能

guix pull が安定

Guix が“普通に使える状態”に到達

## 🧩 補足：この手順のポイント

- WSL では systemd 統合が不完全なため、Guix System のような複雑な統合は不可
- `/gnu` と `/var/guix` を消せば、guix の世界は完全にリセットできる
- 1.5.0 の tarball は WSL で安定して動作する
- substitute の鍵を登録すれば、ビルド地獄を回避し時間節約が大きい（不正な挙動なし）
- `guix pull` が成功すれば最新の Guix が手に入る
- 以降は普通の Linux と同じように使用可能

### WSL 設定（/etc/wsl.conf）

guix pull による systemd 自動起動を有効にするため、以下の設定が必要です：

```ini
[boot]
systemd=true
```

このファイルは Windows 側の `%USERPROFILE%\.wslconfig` と異なり、WSL の Linux インスタンス側の `/etc/wsl.conf` です。

**確認方法：**

```bash
cat /etc/wsl.conf
```

出力が上記の内容であれば、daemon は WSL 起動時に自動起動されます。

## ✨ まとめ

試行錯誤は、WSL における Guix 導入の再現性のある導入手順”を導くためのプロセスになり、その結果を、この Markdown にまとめた。

以下の内容も同じように Markdown 化の候補としていいかも？：

- Emacs + Guix の統合
- Clojure / Babashka 環境
- Guix Home
- Guix チャンネルの設計
- プロファイル戦略
- WSL でのデーモン自動起動