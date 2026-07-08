# JA3MBC 局 FT8/ハムログ連携 調査記録（引き継ぎ用）

## 運用環境
- OS：Ubuntu 24.04
- リグ：Icom IC-7000（USB-シリアル接続 `/dev/ttyUSB0`、CAT制御は Hamlib `rigctld`）
  - CI-Vアドレス：`0x70`（デフォルトの0x76ではない）
  - CI-Vボーレート：リグ側を「自動」から「9600」固定に変更済み（要：rigctld起動時も `-s 9600` を明示）
- WSJT-X：v3.0.0 ab976b（Linuxネイティブ版）
- ログソフト：Turbo HAMLOG/Win Ver5.47c（Wine上で動作）
- 連携ツール：
  - JT-Get's（ハムログ標準搭載、ALL.TXTを表示してクリックでログ転記）← **現在の主運用**
  - JT_Linker（JA2GRC作、UDP/ファイル監視でハムログに自動転記。Wine上で実行）
- 起動スクリプト：`~/ft8_start.sh`（rigctld→WSJT-X→Hamlog→MailQSL→JT_Linkerの順に起動）
  Rust製GUIランチャー「JA3MBC HAM局コントロール」から呼び出し可能

## これまでに解決した問題
1. **CAT制御タイムアウト**：IC-7000のCI-Vボーレートが「自動」だったため`rigctld`との通信が不安定だった
   → 「9600」固定に変更して解決
2. **JT_Linkerの`FindWindow32`エラー（Object reference not set）**：
   - 原因調査の結果、Wine環境ではハムログの「ログ窓A」のウィンドウタイトルが
     全角文字（`ＬＯＧ-[Ａ]`）として生成されており、JT_Linkerが検索する半角の
     `"LOG-[A]"`と一致しないことが直接原因と判明（`wmctrl -l`で確認済み）。
   - `wmctrl`でのタイトル書き換えはWin32 API（FindWindow）には反映されず、解決に至らなかった。
   - `Hamlogw.ini`の`[WSJT]`セクションに`WsjtClass=wsjtx.WSJT-X`を追加してみたが、
     これも症状の改善にはつながらなかった。
   - **最終的に、このエラー自体は実害が薄いことが判明**（下記4参照）。
3. **JT_Linker旧バージョンの整理**：
   - 新バージョン(2024.09.26b)を`~/.wine/drive_c/JT_Linker/`内の旧exe
     （`JT_Linker.exe`、`JT_Linker_2023.exe`）に上書きコピーする形で統一。
   - 不要になった重複インストーラー・フォルダは削除済み。
   - `ft8_start.sh`は`JT_Linker_2023.exe`を呼ぶ設定のまま変更不要で動作。
4. **【最重要】His/My/Freq/Modeが転送されない問題 → 実は解決していた**：
   - 当初、ALL.TXTへの手動追記テストや、JT_Linkerの「Test」ボタン、
     CQ行の誤読などで検証していたが、これらはいずれも
     「WSJT-Xで実際にLog QSOボタンを押した正規のQSO確定データ」ではなかった。
   - 2026/6/19、実際にJG3FSK局とFT8で交信成立し、WSJT-XのLog QSOボタンを押して
     確定したところ、ハムログのログ窓Aに **His・My・Freq・Modeすべて正しく
     自動転送される**ことを確認（His:-16, My:-23, Freq:18.102, Mode:FT8等）。
   - ハムログのメイン一覧（No.1559）にも正しく保存された。
   - **結論：JT-Get's/JT_Linkerの自動転送機能は正常に動作している。**
     これまで「転送されない」と見えていたのは、検証方法（手動データやテストボタン）が
     実際の運用フローと異なっていたために生じた誤認だった。

## 現在進行中の課題
### A. QSO時刻が「開始時刻」で記録される（終了時刻にしたい）
- 症状：JT-Get's/ハムログのTime欄には、CQ応答〜RR73までのやり取りのうち
  **最初の応答時刻**（ALL.TXT上で「Before」マークが付く行）が使われている。
  本来は交信完了（最後のやり取り）の時刻を記録したい。
- WSJT-X自体には開始/終了時刻を明示的に切り替える設定は見当たらなかった
  （Web検索でも、WSJT-X開発者間で同様の議論があるが明確な設定オプションはない様子）。
- 次に確認すべき候補：
  - ハムログ「JT-Get's」画面の「表示(V)」メニュー内に時刻選択に関する設定がないか
  - ハムログのヘルプ(F1)内の「JT-Get's」関連説明
  - `Hamlogw.ini`以外の設定ファイルに時刻関連のキーがないか

### B. 未対応の小課題（次回以降）
- HAMLOG E-Mail QSLの認証エラー（パスワード再設定が必要）
- JT_Linkerの全角ウィンドウタイトル問題自体は未解決のまま
  （実害はないため優先度低だが、気になる場合は別途調査可）

## 参考：技術的な小ネタ
- WSJT-Xのjt9プロセスが孤立して残ることがある → `ps aux | grep jt9` で確認し`kill`すれば解消
- `Hamlogw2.ini`の`[A_HAMLOG]`セクションに、ログ窓Aの「最後にSaveした時のHis/My/Freq/Mode」が
  保存されている。次回ログ窓A起動時の初期値として使われる。

## FreeDV（デジタル音声モード）のオーディオ設定（2026/6/30作業分）
### 構成
- USBオーディオインターフェース（C-Media USB PnP Sound Device）：**リグとの接続専用**
  - ID 52：USB output → リグへの送信音声
  - ID 53：USB input ← リグからの受信音声
- PC本体のオーディオ端子：**人間用（マイク・ヘッドホン）**
  - ID 689：PCI output（`alsa_output.pci-0000_00_1b.0.analog-stereo`）→ 外部ヘッドホンへ
  - ID 690：PCI input（`alsa_input.pci-0000_00_1b.0.analog-stereo.7`）← 外部マイクから
  - ※これらのID番号は環境により変動することがある（再起動等で番号が変わる場合あり）

### 正しい設定（Audio Config画面）
**Receiveタブ：**
- Input From Radio：ID 53（USB input）
- Output To Speaker/Headphones：ID 689（PCI output、外部ヘッドホン）

**Transmitタブ：**
- Input From Microphone：ID 690（PCI input、外部マイク）
- Output To Radio：ID 52（USB output）

### 発生した問題と対処
- 「You must use different devices for To Radio and To Speaker/Headphones」エラー
  → Output To RadioとOutput To Speakerに同じUSBデバイス(ID52)を重複して
    割り当てていたことが原因。PCI側(ID689)とUSB側(ID52)で分離して解決。
- 上記設定後、エラーは解消し「適用」「OK」まで完了。

### 未解決・保留中の事項
- 実際の受信時に「ピー」という音しか聞こえない件
  → FreeDV Reporter画面で確認したところ、他局（JA0DDW→JA7QOU）間では正常に
    SNR12.0等で受信できていたため、ハードウェア設定の問題ではなく
    **その時のコンディション（電波伝搬状況）の問題だった可能性が高い**。
    オーディオ設定自体は完了しているとみてよい。
- 「Start Recording」（録音機能）画面が操作中に出てきたが、これは本題と直接関係なく、
  単にコンディション不良でデコードできず色々触っていた際に開いた様子。深追いはしていない。
  必要であれば次回、録音機能（Off Air/Decoded、WAV/MP3選択）について改めて確認する。
