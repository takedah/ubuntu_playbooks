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

- 手元に Ansible（core 2.15 以降。`deb822_repository` モジュールを使う）
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
