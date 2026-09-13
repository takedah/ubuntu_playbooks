# Ubuntu playbooks

## Description

Ubuntu 26.04 LTS に開発環境を構築する Ansible Playbook。
さくらのクラウドの「Ubuntu Server 26.04 LTS 64bit」パブリックアーカイブを対象にしている。

構築されるもの:

| | |
|---|---|
| ランタイム | mise 経由の Python / Node.js / uv |
| エディタ | Neovim（公式ビルド済みバイナリ）+ vim-plug + coc.nvim |
| ターミナル | tmux |
| コンテナ | Docker Engine + Compose plugin |
| 設定 | [takedah/dotfiles](https://github.com/takedah/dotfiles) を clone してシンボリックリンク |

Neovim と tmux の設定は dotfiles リポジトリが正となる。
このリポジトリは設定を複製せず、リンクを張るだけ。

## Requirements

- 手元に Ansible（core 2.17 以降。`deb822_repository` と `systemd_service` を使う）
- 対象サーバへ公開鍵で SSH できること
  さくらのクラウドの通常アーカイブには cloud-init が入っていないので、
  公開鍵はサーバ作成時にコントロールパネルで投入しておく
- `~/.ssh/config` に対象ホストのエイリアスを書いておく

```
Host dev
  HostName <サーバのグローバル IP>
  User ubuntu
  IdentityFile ~/.ssh/id_ed25519
```

## Usage

sudo のパスワードが要る場合は `--ask-become-pass` を付ける。
さくらのクラウドの `ubuntu` ユーザは NOPASSWD なので通常は不要。

```console
$ ansible-playbook -i develop dev.yml
```

2回目以降の差分確認には `--check --diff` が使える。

```console
$ ansible-playbook -i develop dev.yml --check --diff
```

まっさらなサーバに対する `--check` は途中で失敗する。
check モードでは apt が実際には走らないため、
そこで入るはずのコマンド（mise・nvim・batcat など）に依存する
後続タスクが軒並みこけるためで、これは Ansible の check モードの
性質によるもの。初回はそのまま本番実行してよい。

ローカルの Ubuntu に対して流す場合:

```console
$ ansible-playbook -i local local.yml --ask-become-pass
```

## After the first run

1. 再ログインする（`docker` グループへの追加と `.profile` の PATH を反映させるため）
2. `nvim` を対話的に一度起動する
   `g:coc_global_extensions` に並べた coc 拡張が自動でインストールされる。
   headless での `CocInstall -sync` は不安定なので Playbook では実行していない
3. `:checkhealth` で provider の状態を確認する
   `python3` と `node` が OK、`ruby` / `perl` は disabled になっていれば想定どおり

## VNC デスクトップ

開発サーバに XFCE デスクトップを立て、手元の VNC クライアントから操作する。
常用する想定ではないので `dev.yml` には入れず、必要なときだけ流す。

```console
$ ansible-playbook -i develop vnc.yml
```

VNC パスワードを対話で聞かれる。TigerVNC の VncAuth は **8 文字を超える分を
捨てる**ので、9 文字目以降を付けても意味がない。ログインパスワードの
使い回しは避けること。

### 接続する

VNC は `127.0.0.1:5901` でしか待ち受けない。SSH トンネルを張って繋ぐ。

```console
$ ssh -L 5901:localhost:5901 dev
```

トンネルを張ったまま、別の端末から VNC クライアントを `localhost:5901` へ
向ける。macOS なら標準の画面共有でよい。

```console
$ open vnc://localhost:5901
```

### 構成

| | |
|---|---|
| デスクトップ | XFCE（`vnc_desktop_packages`） |
| ブラウザ | Firefox（Mozilla 公式リポジトリの deb、`vnc_browser_packages`） |
| 日本語入力 | fcitx5 + Mozc（切り替えは Ctrl+Space） |
| ディスプレイ番号 | `:1` = TCP 5901（`vnc_display`） |
| 解像度 | 1920x1080（`vnc_geometry`） |
| 待ち受け | ループバックのみ |
| サービス | `tigervncserver@:1.service` |
| 設定ディレクトリ | `~/.config/tigervnc`（`~/.vnc` は TigerVNC 1.13 以前の旧パス） |

ループバック限定は `/etc/tigervnc/vncserver-config-mandatory` に書いている。
このファイルはユーザの `~/.config/tigervnc/tigervnc.conf` とコマンドラインの**両方を
上書きする**ので、設定ミスで LAN に露出することがない。

XFCE は Recommends を切って入れている。残すと `lightdm` と `xserver-xorg` が
付いてきて 150 → 460 パッケージに膨らむが、ヘッドレスの VM ではどちらも
不要（X サーバは Xtigervnc 自身が担い、コンソールにログイン画面は要らない）。

### ブラウザ

Ubuntu の `firefox` パッケージは snap を入れるだけの transitional パッケージ
（`1:1snap1-...`、"Installs Firefox snap"）で、さくらのクラウドのアーカイブに
snapd は含まれない。`chromium` に至っては archive にすら無い。そのため
Mozilla 公式の apt リポジトリから実体の deb を入れている。

Ubuntu 側の `firefox` はエポック `1:` を持つので、APT ピンを置かないと
Mozilla の `155.0.1~build1` より上位に並んで snap 版が選ばれてしまう。
`/etc/apt/preferences.d/mozilla` で `firefox*` だけを優先している。
このリポジトリには `firefox*` と `mozillavpn` しか無いため、Mozilla が案内する
`Package: *` より狭く絞ってある。

XFCE の「Web Browser」ランチャーは `exo-open --launch WebBrowser` を呼ぶ。
ブラウザが一つも入っていないと `debian-sensible-browser` が選ばれ、
`sensible-browser` が最終手段の `www-browser` を起動しようとして

```
Failed to execute child process "www-browser": Failed to execve: No such file or directory.
```

になる。`~/.config/xfce4/helpers.rc` に `WebBrowser=firefox` を書くことで
`sensible-browser` を経由せず直接起動させている。

設定 → 「既定のアプリケーション」から別のブラウザに変えることもできるが、
Playbook を流し直すと `WebBrowser=firefox` に戻る。恒久的に変えるなら
`vnc_browser_packages` と合わせてロール側を直すこと。

### 日本語入力

fcitx5 + Mozc。英語配列キーボードで半角/全角キーが無いため、切り替えは
fcitx5 既定の **Ctrl+Space** をそのまま使う。非アクティブ時は
`keyboard-us`、アクティブ化すると Mozc に入る。

> Neovim の `init.vim` は `<c-space>` を `coc#refresh()` に割り当てている。
> VNC デスクトップ内の端末で nvim を使うと、Ctrl+Space は fcitx5 に
> 先に取られる。変えるなら `fcitx5-config-qt` の「Global Options」から。

セットアップで踏んだ罠を2つ記録しておく。どちらも「パッケージを入れても
日本語が打てない」で終わる類のもの。

**im-config の auto モードは英語ロケールで効かない**
`im-config` の auto は、デスクトップが `CJKV_DEFAULT_DESKTOP` に載っていて
ロケールが CJKV でない場合に `none` を返す
（`/usr/share/im-config/xinputrc.common` の `echo_cjkv_selected_im`）。
そのため `~/.xinputrc` に `run_im fcitx5` と明示している。
これが無いと `/etc/X11/Xsession.d/70im-config_launch` が何も起動しない。

**fcitx5 の自動グループ生成に Mozc が入るのは LANG が日本語のときだけ**
fcitx5 は有効なグループが無いと起動時に自動生成するが、その中身は
ロケールで変わる。

| ロケール | 生成されるグループ |
|---|---|
| `LANG=C.UTF-8` | `keyboard-us` のみ |
| `LANG=C.UTF-8 LC_CTYPE=ja_JP.UTF-8` | `keyboard-us` のみ |
| `LANG=ja_JP.UTF-8` | `keyboard-us`, `mozc` |

`keyboard-us` だけのグループでは Ctrl+Space を押しても切り替わる先が無い。
UI 言語を英語のままにしたいので、`~/.config/fcitx5/profile` にグループを
書いて LANG に依存させていない。

この profile は `force: false` で置いている。`fcitx5-config-qt` で入力
メソッドを足したり既定を変えたりした結果を、流し直しで巻き戻さないため。
初期状態からやり直したいときは消してから流す。

```console
$ ssh dev rm .config/fcitx5/profile
$ ansible-playbook -i develop vnc.yml
```

`ja_JP.UTF-8` ロケールは生成してあるので、デスクトップごと日本語にしたい
場合は `LANG` を切り替えればよい。

### パスワードを変える

`~/.config/tigervnc/passwd` は難読化された 8 バイトで、平文を持っていない
ため差分が取れない。初回だけ作る作りにしてあるので、変更するときは消して
から流し直す。

```console
$ ssh dev rm .config/tigervnc/passwd
$ ansible-playbook -i develop vnc.yml
```

## 旧版で構築済みのホストへ適用するとき

旧版の Playbook を流したことがあるホストには、Playbook では消せない
残骸がある。Playbook が自動で片付けるのは旧 `nvim` バイナリと
旧 `docker.list` の2つだけなので、以下は手作業で確認する。

**dotfiles が更新されない**
`dotfiles` ロールは `update: false`（ホスト上での編集を巻き戻さないため）
なので、clone 済みだと古いままになる。旧 `.profile` は pyenv / rbenv / nvm を
読み込むため、ログインのたびにエラーが出て node にも PATH が通らない。

```console
$ git -C ~/dotfiles pull
```

**ソースビルドした tmux が優先される**
旧版は tmux を `/usr/local/bin/tmux` にビルドしていた。apt 版は
`/usr/bin/tmux` に入るが PATH では `/usr/local/bin` が先なので、
古いほうが使われ続ける。エラーにならないので気づきにくい。

```console
$ which tmux            # /usr/local/bin/tmux なら旧ビルド
$ sudo rm /usr/local/bin/tmux
```

**coc 拡張が自動で入らない**
`~/.config/coc/extensions/package.json` に拡張が記載済みだと coc は
導入済みと判断して何もしない。実体が消えていると、拡張が無いまま
自動インストールも走らない状態になる。

```console
$ rm -rf ~/.config/coc
$ nvim                  # 起動後、非同期で13個入る
```

## Notes

- Ubuntu 26.04 は sudo-rs と Rust 版 coreutils が既定。
  `sudo` のワイルドカード引数ルールが効かない、`sort` / `split` の挙動が
  GNU 版と細部で異なるなどの非互換がある
- パブリックアーカイブは swap 領域を持たない。メモリの小さいプランで
  プラグインのインストールが OOM する場合は事前に swap ファイルを作る
- tmux のクリップボード連携は OSC 52 経由になる。
  手元のターミナルが OSC 52 に対応している必要がある
  （iTerm2 / WezTerm / Ghostty / Alacritty は対応、Terminal.app は非対応）

## Versions

バージョンは `group_vars/all.yml` で固定している。上げるときはここを編集する。

Neovim と vim-plug はチェックサムでも固定しているので、
バージョンを上げるときは `neovim_checksums` / `vim_plug_checksum` も一緒に更新する。

```console
$ curl -sfL https://github.com/neovim/neovim/releases/download/<ver>/nvim-linux-x86_64.tar.gz | shasum -a 256
$ curl -sfL https://github.com/neovim/neovim/releases/download/<ver>/nvim-linux-arm64.tar.gz  | shasum -a 256
$ curl -sfL https://raw.githubusercontent.com/junegunn/vim-plug/<ver>/plug.vim | shasum -a 256
```

チェックサムが合わないと `get_url` がその場で失敗するので、
配布物が差し替わったことに気づける。
