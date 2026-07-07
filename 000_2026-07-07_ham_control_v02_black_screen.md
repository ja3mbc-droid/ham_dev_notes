# 2026-07-07 作業ログ: ham_control_v02 黒画面問題の調査と解決

## 経緯

`ham_control`(本体・使用版)の開発状況レビューから開始し、GitHub運用の整理、そして
開発中の `ham_control_v02`(flrig直接連携版)における「ウィンドウは開くが中身が
真っ黒のまま表示されない」不具合の調査・解決を行った。

---

## 1. GitHubリポジトリ整理

### 発見した2つの新規リポジトリ
- **`ham_control_0.1`**: `ham_control` の完全な複製(コミット1つ)。
  中身は`ham_control`と一字一句同一だった。
- **`fldigi`**: W1HKJ/fldigi のフォーク。null pointerセグフォルト修正を
  upstreamにIssue報告済み(以前のセッションで対応)。

### 方針決定
- `ham_control_0.1` は「安定版(main)を壊さずに開発したい」という意図で
  作成されたことが判明。ただし別リポジトリでの複製は
  - 中身の同期漏れに気づきにくい
  - 履歴が分断される
  - 開発版→使用版への「昇格」作業が手動コピーになりミスが起きやすい
  という問題があるため、**同一リポジトリ内でのブランチ運用
  (例: `main` = 使用版 / `dev` = 開発版)への移行を推奨**。
  ただし今回は「ローカルで作業途中・時間切れで中途半端」という背景があり、
  即移行はせず今後の課題として保留。

### GitHubプロフィールの整理
- Pinned repositories を `ham_control`(本体・継続コミットあり)と
  `w1hkj/fldigi`(★163・フォーク39のupstream実績が可視化される)に設定。
  `ham_control_0.1` や `hello_rust` など見せる優先度の低いものは外した。

---

## 2. ham_control_v02: 黒画面問題の切り分けと解決

### 症状
`cargo run` で起動すると、タイトルバーは表示されるが本体(描画領域)が
真っ黒のまま固まったように見える。

### 切り分け手順と結果

| 確認項目 | 結果 |
|---|---|
| flrigは起動しているか | Yes(curlで`rig.get_vfo`に対し `7086000` の応答を確認、通信は正常) |
| プロセスはクラッシュしているか | No(`cargo run` はエラーなく `Running` 状態を継続) |
| `LIBGL_ALWAYS_SOFTWARE=1` で起動 | **表示された**(`RAW VALUE: 7086000` / `Frequency: 7.086000 MHz` 等が正常表示) |
| `glxinfo` でGPU情報確認 | `Mesa Intel(R) HD Graphics (ILK)` / OpenGL 2.1(coreプロファイル非対応、compatのみ) |
| `eframe::Renderer::Glow` 指定を試す | 改善せず(黒画面のまま) |

### 原因の特定
N5010搭載のIntel HD Graphics(Ironlake世代、2010年前後)は **OpenGL 2.1・
coreプロファイル非対応** という古いGPUであり、eframe(egui)のハードウェア
レンダリングパス(wgpu/Glowいずれも)との相性が悪く、描画が完了しない状態に
陥っていた。`LIBGL_ALWAYS_SOFTWARE=1` によりMesaのソフトウェアラスタライザ
(llvmpipe)へフォールバックさせることで解決した。

### 対処内容
1. `~/ham_control_v02_start.sh` を作成し、`LIBGL_ALWAYS_SOFTWARE=1` を
   自動的に付与した上で `cargo run --release` を実行するようにした。
2. `ui.rs` に `renderer: eframe::Renderer::Glow` を明示追加
   (実際の解決には寄与しなかったが、保険として残置)。

---

## 3. 副次的な発見: ham_control本体にも同じ問題が潜在

`ham_control` 本体を `cargo run` で直接デバッグ起動したところ、
**同じ黒画面現象が再現**した。ただし調査の結果、普段使用している
サイドバーのランチャーアイコン(`~/.local/share/applications/ham_control.desktop`)
には、**既に `LIBGL_ALWAYS_SOFTWARE=1` が仕込まれていた**ことが判明。

```
Exec=bash -c 'LIBGL_ALWAYS_SOFTWARE=1 /home/ja3mbc/ham_control/target/release/ham_control'
```

→ 新規のバグではなく、**過去に同じ問題へ対処済みだったことの再発見**であり、
実運用上は問題が起きていなかったことを確認できた。

---

## 4. 実施した変更まとめ

### `ham_control` リポジトリ (main ブランチ, commit `c7c6501`)
- 13番目のボタン「HAM CONTROL v02 起動 (flrig周波数モニタ)」を追加
  (`ham_control_v02_start.sh` を呼び出す)
- ウィンドウサイズを `[320.0, 520.0]` → `[340.0, 640.0]` に拡大
  (ボタン数増加により画面下部が見切れる問題への対応)
- `renderer: eframe::Renderer::Glow` を追加(保険的措置)

### `ham_control_v02` リポジトリ (master ブランチ, commit `e0d9aed`)
- `renderer: eframe::Renderer::Glow` を追加(保険的措置)

### ローカル環境(N5010)
- `~/ham_control_v02_start.sh` を新規作成・実行権限付与
- `~/.local/share/applications/ham_control_v02.desktop` を新規作成
  (サイドバーから単独起動できるように)

---

## 5. 今後の課題(未着手)

- `ham_control_v02` の `config.rs` / `rig.rs` / `wsjtx.rs` / `hamlog.rs` は
  まだ空のモジュール(スタブ)のまま
- `flrig.rs` の通信がブロッキング(同期)呼び出しのため、将来的にflrig応答が
  遅延した場合はGUIがフリーズするリスクが残っている(別スレッド化を推奨)
- `Client::new()` を毎回生成している非効率(使い回しへの改善余地)
- XML応答のパースが文字列分割による簡易実装(堅牢性に課題)
- `ham_control_0.1` の扱い(削除するか、ブランチ運用へ移行するか)は保留中
