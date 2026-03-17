
# 🧭 WSL における Guix 1.5.0 導入手順

目的：WSL 上に Guix のパッケージ管理機能を備えた環境（Guix 1.5.0）を構築。
前提：WSL2（Ubuntu）、root 権限

## 📌 この手順が対象とするもの

- **Guix System のインストール** ❌ ではない
- **Guix を Debian/Ubuntu と同じパッケージマネージャとして使う** ✅
- **Guix System のような完全な systemd 統合は想定していない手順** ✅
- デーモンの自動起動までは対象外（必要に応じて wrapper script を作成）

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

## 4. デーモンを起動（WSL は手動起動）

展開されたディレクトリを確認：

```bash
ls /gnu/store/ | grep guix
```

このコマンドで `xxxxxxx-guix-1.5.0` 形式のディレクトリが見つかります。以下でデーモンを起動：

```bash
/gnu/store/*-guix-1.5.0/bin/guix-daemon --build-users-group=guixbuild
```

WSL では systemd 統合が不完全なため、デーモンは手動で起動する必要がある。

### 4.1 日々のデーモン管理

**デーモンは root で起動・保持**

デーモンが起動したままの状態を保つ必要があり、以下の方法から選択：

**方法1: 毎回手動で起動**

Guix を使う度に以下を実行：
```bash
sudo /gnu/store/*-guix-1.5.0/bin/guix-daemon --build-users-group=guixbuild &
```

**方法2: シェル起動時に自動実行（推奨）**

`/home/mtok/.bashrc` に以下を追加：
```bash
pgrep -f guix-daemon > /dev/null || sudo /gnu/store/*-guix-1.5.0/bin/guix-daemon --build-users-group=guixbuild &
```

**注意**: パスワードなしで sudo を実行するため、`/etc/sudoers` に以下を追加して置いてください：
```
mtok ALL=(ALL) NOPASSWD: /gnu/store/*-guix-*/bin/guix-daemon
```

**方法3: systemctl で管理（参考：WSLでは不安定らしい）**

systemd が利用可能であれば、以下でデーモンを起動：
```bash
sudo systemctl start guix-daemon
```

起動時に自動起動させたい場合：
```bash
sudo systemctl enable guix-daemon
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

**一般ユーザーから guix を使う場合**

まず、必要なディレクトリを作成し、ユーザープロファイルを初期化します：

```bash
mkdir -p ~/.config/guix
/gnu/store/*-guix-1.5.0/bin/guix pull
```

初期化後、各ユーザーのシェル起動スクリプト（`~/.bashrc` など）に以下を追加：
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

**注意**: 
- デーモンは root で起動・保持されているため、一般ユーザーは guix コマンド経由で Guix 機能を利用します
- 各ユーザーは独立した `~/.config/guix/current` を持つため、それぞれで `guix pull` を実行する必要があります

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

## ✨ まとめ

試行錯誤は、WSL における Guix 導入の再現性のある導入手順”を導くためのプロセスになり、その結果を、この Markdown にまとめた。

以下の内容も同じように Markdown 化の候補としていいかも？：

- Emacs + Guix の統合
- Clojure / Babashka 環境
- Guix Home
- Guix チャンネルの設計
- プロファイル戦略
- WSL でのデーモン自動起動