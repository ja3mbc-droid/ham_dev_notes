# 2026-07-07 作業ログ: ham_dev_notes リポジトリ立ち上げと main/master 統一

## 経緯

前回の記録(`000_2026-07-07_ham_control_v02_black_screen.md`)を、今後も継続的に
残していくための専用リポジトリ `ham_dev_notes` を新設した。その過程で、
GitHubのデフォルトブランチ名 (`main` / `master`) の食い違いに気づき、解消した。

---

## 1. ham_dev_notes リポジトリ新設

- 複数プロジェクト(`ham_control`, `ham_control_v02`, `fldigi`等)に
  またがる開発記録を1箇所に集約する目的で新設。
- ファイル命名規則を決定: **`連番3桁_YYYY-MM-DD_短いタイトル.md`**
  - 例: `000_2026-07-07_ham_control_v02_black_screen.md`
  - 記念すべき初回のため `000` からスタート(次回以降は `001`, `002`...)
  - 3桁で約6〜7年分の記録に対応可能と判断(週2〜3回ペース想定)

---

## 2. main / master ブランチの混乱と解消

### 発生した問題

`git init` でローカルにリポジトリを作成すると、Gitのバージョンによっては
デフォルトで **`master`** という名前のブランチが作られる。一方、GitHub側で
`curl` によりAPI経由でリポジトリを作成すると、レスポンスには
`"default_branch": "main"` と返ってくる。

この状態で `git push -u origin master` を実行すると、ローカルの内容は
`master`ブランチとしてGitHubに届くが、**GitHub側が期待する `main` ブランチとは
一致しない**、という食い違いが生じる。

今回は幸い、`main`ブランチ自体はまだ実体として作られていなかった
(`git ls-remote --heads origin` で確認したところ `refs/heads/master` のみ存在)
ため実害はなかったが、放置すると将来「`main`ブランチを見たら空だった」という
混乱を招く可能性があった。

### 実施した対応

1. リポジトリ間でブランチ名がばらついていることを確認
   - `ham_control`: `main`
   - `ham_control_v02`: `master`
   - `ham_dev_notes`: `master`(このままでは不統一)
2. `ham_dev_notes` を `main` に統一する方針とした
3. ローカルで一旦 `main` ブランチを作成しpush:
   ```bash
   git branch -m master main
   git push -u origin main
   ```
4. GitHub側の **Settings → General → Default branch** で、デフォルトブランチを
   `master` → `main` に変更(切り替え用のアイコン⇄を使用。隣接する鉛筆アイコンは
   「Rename branch」なので誤って押さないよう注意)
5. デフォルトブランチ変更後、空になった `master` を削除:
   ```bash
   git push origin --delete master
   ```

### 再発防止策

ローカルGitの初期設定を変更し、今後 `git init` する際は最初から `main` に
なるようにした:

```bash
git config --global init.defaultBranch main
```

これにより、`ham_control_v02` のように後から気づかず `master` のまま
放置される事態を防止できる。

---

## 3. 学んだこと(今後のためのメモ)

- GitHub API経由でのリポジトリ作成時の `default_branch` 表示は、実際にpush
  されるまでは「予約された名前」に過ぎず、ローカルの `git init` 時のデフォルト
  ブランチ名と食い違うことがある
- ブランチの「Rename」と「Switch(切り替え)」はGitHub UI上で似た位置に
  アイコンが並んでいるため、押し間違いに注意(鉛筆=Rename、⇄=Switch)
- 複数リポジトリを横断的に運用する場合、ブランチ名の統一(`main`推奨)は
  早い段階でしておいた方が、後々の混乱を避けられる
