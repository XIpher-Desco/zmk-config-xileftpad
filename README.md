# zmk-config-xileftpad

左手用キーパッド **xileft pad**（27キー + ロータリーエンコーダ 3個）の ZMK ファームウェア設定。

QMK/Vial 版 `xipher/xileft_pad_v3` からの移行。コントローラは RP2040-Zero から
**Seeed XIAO nRF52840 Plus** に変更、無線（BLE）対応。

> 実機を使うための手順（書き込み・Bluetooth 接続・キーマップ変更）は
> **[docs/MANUAL.md](docs/MANUAL.md)** にまとめてある。この README は設計・開発向けの詳細。

---

## ファイル構成

| パス | 内容 |
| --- | --- |
| `config/xileft_pad.keymap` | **編集するのはここ。** キーマップ本体（6レイヤ + エンコーダ） |
| `config/xileft_pad.conf` | 機能の ON/OFF（エンコーダ、マウス、電池、省電力） |
| `config/west.yml` | ZMK 本体の参照先 |
| `boards/shields/xileft_pad/xileft_pad.overlay` | 配線定義（マトリクス、エンコーダ、ピン競合の回避） |
| `boards/shields/xileft_pad/xileft_pad-layouts.dtsi` | ZMK Studio 用の物理レイアウト（キー座標） |
| `boards/shields/xileft_pad/xileft_pad.keymap` | シールド単体用の最小デフォルト。**通常は使われない** |
| `build.yaml` | ビルド対象。ZMK Studio 有効 |
| `config/xileft_pad_ipad.keymap` | iPad 版キーマップ（マウス機能なし） |
| `config/xileft_pad_ipad.conf` | iPad 版の機能 ON/OFF（`CONFIG_ZMK_POINTING` を入れない） |
| `boards/shields/xileft_pad_ipad/` | iPad 版シールド。overlay は通常版を include するだけ |

## ビルドターゲット

| シールド | 基板 | 接続先 | マウス | デバイス名 |
| --- | --- | --- | --- | --- |
| `xileft_pad` | v1 | PC | あり | `xileft BLE` |
| `xileft_pad_ipad` | v1 | iPad | **なし** | `xileft iPad` |
| `xileft_pad_v2` | v2 | PC | あり | `xileft BLE v2` |
| `xileft_pad_v2_ipad` | v2 | iPad | **なし** | `xileft iPad v2` |

iPad 版は `CONFIG_ZMK_POINTING` を入れず、`&mkp` / `&msc` を一切使わない。
そのため keymap では BASE row 3 左端が `&mo 1`（レイヤ切替のみ）、
エンコーダ2が `Ctrl` + テンキー `+/-`（クリスタのズーム）になっている。

配線定義は通常版の overlay を `#include` しているだけなので、**ピン割り当ての
修正は必ず通常版側だけを直すこと**。キーマップも v2 版は v1 版を include している。

---

## ビルドと書き込み

```sh
git add .
git commit -m "..."
git push
```

push すると GitHub Actions が走り、Artifacts から `xileft_pad.uf2` を取得できる。

書き込みは **XIAO のリセットボタンを素早く2回押す** → USB ドライブとして
マウントされるので、そこに `.uf2` をコピー。

> RP2040 用の `.uf2` とは互換性がないので取り違えに注意。

---

## 最初にやること（重要）

### 0. v1 基板のリワーク（電池残量を表示するために必要）

ROW5 は回路図上のラベルのみでスイッチ実体が無い（このキーボードは 27 キー）。
そのため **D12（P0.19、パッド16）がパターン上空いている**。これを利用して、
電池電圧読み取りピンに載ってしまっている ROT3 を移設する。
**カット1箇所＋ジャンパ1本**で完了する。

| 手順 | 作業 | 場所 |
| --- | --- | --- |
| ① カット | ROT3（エンコーダ2の B 相）→ D16 のパターンを切断 | **D16（P0.31）パッドの近く**で切る |
| ② ジャンパ | エンコーダ2の B 相（カットのエンコーダ側）→ **D12 パッド（P0.19）** | ROW5 用だったが未使用のパッド |

D12（P0.19）は BSP 全ソースでピン番号が一致している安全なピンなので、
新たな曖昧さは持ち込まない。①で P0.31 が完全に空き、
**電池残量レポート（BLE の電池表示）が有効になる**。

リワーク後の確認: テスターで「P0.31 パッド ⇔ エンコーダ2の B 脚」が
**導通していない**こと（カット完了）、「D12 ⇔ B 脚」が導通していること。
カットが不完全だと電池残量が 100% や 0% 付近に張り付くので、
その症状が出たらカット箇所を再確認。

※ リワークせずに使う場合は、`xileft_pad.overlay` の `encoder_2` の
`b-gpios` を `<&gpio0 31 ...>` に戻し、`&vbatt { status = "disabled"; };` と
`CONFIG_ZMK_BATTERY_REPORTING=n` を復活させること（電池残量なしで動作する）。

### 1. ROT4 のピン番号を実測で確定する

Seeed の BSP 内で D17 / D19 のポート対応が食い違っている。現在は
`&gpio1 3`（P1.03）を仮採用している。

**エンコーダ3だけが反応しない場合**、`xileft_pad.overlay` の `encoder_3` の
`a-gpios` を `<&gpio1 7 ...>`（P1.07）に変更する。

### 2. エンコーダの回転方向を確認する

回転方向が逆なら、該当エンコーダの `a-gpios` と `b-gpios` を入れ替える。
QMK 版ではエンコーダ1のみ A/B が逆順に定義されていたため、ここは特に要確認。

### 3. エンコーダの分解能を調整する

`overlay` の `triggers-per-rotation` は QMK の `resolution` からの換算値。
1ノッチで反応しすぎる／足りない場合はここを増減する。

| | QMK resolution | triggers-per-rotation |
| --- | --- | --- |
| encoder 1 | 4 | 20 |
| encoder 2 | 2 | 40 |
| encoder 3 | 2 | 40 |

### 4. 動作確認は USB → BLE の順で

先に USB 接続で全キー・全エンコーダを確認する。無線から始めると、
配線の問題か無線の問題かの切り分けができなくなる。

### 5. 動作確認が済んだら ZMK のバージョンを固定する

`config/west.yml` は ZMK の `main` を追従している（`xiao_ble//zmk` 命名と
Zephyr 4.1 の機能が必要なため）。動く状態を確認したら、`revision: main` を
その時点のコミット SHA に書き換えて凍結すること。main のままだと
ZMK 側の変更でビルドが突然壊れることがある。

---

## Bluetooth の使い方

**BASE レイヤの row 2、`U` の右隣のキーを押している間だけ BT レイヤになる。**

| BT レイヤ内 | 動作 |
| --- | --- |
| row 0 の 4 キー | 接続先プロファイル 1〜4 に切り替え（`BT_SEL 0-3`） |
| row 1 左端 | プロファイル 5（`BT_SEL 4`） |
| row 1 の 2 番目 | **現在のプロファイルのペアリング情報を消去**（`BT_CLR`）。接続をやり直すときに使う |
| row 1 右の 2 キー | 出力先を USB / BLE に強制切り替え（`OUT_USB` / `OUT_BLE`） |

初回ペアリング: プロファイルを選ぶ → PC/スマホの Bluetooth 設定から
「xileft BLE」を追加。つながらないときは該当プロファイルで `BT_CLR` して
デバイス側の登録も削除し、やり直すのが確実。

---

## ステータス LED（zmk-rgbled-widget）

XIAO オンボードの RGB LED（赤 P0.26 / 緑 P0.30 / 青 P0.06）で、電池残量と
BLE 接続状態を**色**で示す。外部モジュール
[caksoylar/zmk-rgbled-widget](https://github.com/caksoylar/zmk-rgbled-widget)
を `config/west.yml` から取り込んでいる。

| タイミング | 表示 |
| --- | --- |
| 起動時 | 電池レベル色を約8秒 → 接続状態色を約1秒（合計 約9.5秒）。これが電源投入の合図を兼ねる |
| BT プロファイル切替時 | 接続状態色を約1秒（自動。キーマップに書く必要はない） |
| BT レイヤ row 1 中央（`&ind_bat`） | 電池レベル色 |
| BT レイヤ row 2 左端（`&ind_con`） | 接続状態色 |
| 残量 5% 未満 | 赤で定期的に点滅 |

色の意味は 電池: 緑=80%以上 / 黄=20〜80% / 赤=20%未満、
接続: 青=接続済 / 黄=ペアリング待ち / 赤=未接続。しきい値と色は
`CONFIG_RGBLED_WIDGET_*` で変更できる。

### 設計上の制約（変更前に読むこと）

- **輝度は変えられない。** モジュールは Zephyr の `gpio-leds` を要求し
  `led_on()` / `led_off()` しか呼ばないため、PWM 化しても効かない。
  実質の調整つまみは点灯時間だけ
- **点灯時間は用途ごとに分けられない。** `BATTERY_BLINK_MS` は
  「起動時」と「残量表示キー」で共通、`CONN_BLINK_MS` は
  「起動時」と「BT 切替時」で共通
- 以前ここにあった `EXT_POWER`（緑 LED 常時点灯）は、同じ P0.30 を
  widget と取り合うため撤去した。常時点灯が要るなら widget 側を諦めること
- モジュールは ZMK 本体とバージョンを合わせる必要がある。ZMK を
  コミット SHA で固定するときは、**このモジュールも同時に固定する**

---

## ハードウェア上の注意

このキーボードは XIAO nRF52840 Plus の GPIO を 17 本使っている
（COL 6 + ROW 5 + ROT 6）。overlay には以下の対処が入っている。

| 対処 | 理由 |
| --- | --- |
| `&uicr { nfct-pins-as-gpios; }` | ROT1/ROT2 が NFC 兼用ピン（P0.09 / P0.10）に載っているため |
| `&xiao_i2c` / `&xiao_spi` を disabled | 既定の pinctrl が COL4/5・ROW1-4 と重なるため（将来の衝突予防） |

`nfct-pins-as-gpios` は UICR に一度だけ書き込まれる。**元に戻すにはチップの
全消去が必要**なので注意（実用上の問題はない）。

電池はリワーク（P0.31 の解放）により**残量レポート込みで動作する**。
接続先のデバイスから BLE の電池残量として見える。

---

## ZMK Studio（実機でのキーマップ編集）

`build.yaml` で有効化済み。<https://zmk.studio/> から USB 接続で使える。

- アンロックキーは **LOWER レイヤの左上**（`&studio_unlock`）
- LOWER へは BASE の `&lt_mkp 1 MB2`（元 `LT(1, KC_MS_BTN2)`）を長押しで遷移

> **注意**: Studio で一度キーマップを編集すると、以降 `.keymap` ファイルを
> 書き換えても反映されなくなる。ファイル側の編集を再び有効にするには
> Studio で「Restore Stock Settings」を実行すること。

Studio ではエンコーダの回転動作を割り当てられない（ZMK 側が未対応）。
エンコーダは `config/xileft_pad.keymap` の `sensor-bindings` で管理する。

---

## QMK/Vial から落とした機能

| 機能 | 理由 |
| --- | --- |
| RGB（WS2812 24個 / VialRGB） | ZMK の underglow はエフェクトが4種のみ。電池運用では消費電力も厳しい |
| ゲームパッド（BTN00–BTN31） | ZMK に HID ゲームパッド機能がない |
| OS Detection | 相当機能なし |

これらが割り当てられていた位置は `&none` にしてある。旧 LIGHT レイヤ
（index 3）は **BT レイヤ**に変更した（上記「Bluetooth の使い方」参照）。

基板の `LED` ネット（D6 / P1.11）は未使用（WS2812 未実装）。
ROW5 ラベルはスイッチ実体が無いため、ファームでは行ごと定義していない。

---

## 既知の注意点

- `CONFIG_ZMK_POINTING`（マウスボタン / ホイール）の有効・無効を後から
  切り替えると HID ディスクリプタが変わるため、**BLE 接続済みのデバイスでは
  マウス系が効かなくなることがある**。その場合は BT レイヤの `BT_CLR` と
  デバイス側の登録削除で一度ペアリングを消し、繋ぎ直すこと。
- 電池残量が 100% や 0% 付近に張り付く場合は、リワークのカット（ROT3 → D16）が
  不完全で P0.31 がエンコーダ線につながったままの可能性が高い。
- **`&msc`（マウススクロール）をエンコーダに割り当てるときは `tap-ms` と速度値の
  両方を明示すること。** `&msc` は「押している間ずっとスクロールし続ける」時間ベースの
  behavior で、16ms ごとに `速度 x 0.016` を積算する方式。sensor-rotate の
  デフォルト `tap-ms = 5` だと最初の tick が来る前に離すため、**配線が正常でも
  スクロールが1回も出ない**。`config/xileft_pad.keymap` の `inc_dec_msc` の
  コメントに算出根拠あり。`&mmv`（カーソル移動）も同じ仕組みなので同様の注意が要る。

---

## v2 ― 次回発注基板のピン割り当て

`xileft_pad_v2` シールドは、次に発注する基板向けの改訂版（27 キー）。
ROW5 の廃止で必要ピンが 17 本になり、**NFC ピンも電池ピンも一切使わない
妥協ゼロの設計**が成立する。**回路図・シルクは D 番号ではなく P 番号
（P0.xx / P1.xx）で管理すること**（Seeed の D17/D19 表記はソースによって
食い違いがあるため）。

| ネット | nRF52840 | 参考 D 番号 | 備考 |
| --- | --- | --- | --- |
| COL0–COL5 | P0.02 / P0.03 / P0.28 / P0.29 / P0.04 / P0.05 | D0–D5 | v1 と同じ |
| ROW0–ROW4 | P1.11 / P1.12 / P1.13 / P1.14 / P1.15 | D6–D10 | D6 を LED からマトリクスに転用 |
| ENC1 A/B | P0.15 / P0.19 | D11 / D12 | |
| ENC2 A/B | P1.01 / P1.03 | D13 / SCK1 | |
| ENC3 A/B | P1.05 / P1.07 | MISO1 / MOSI1 | |
| **未接続** | **P0.31** | D16 (BAT) | **空けることで電池残量レポートが標準で機能** |
| **未接続** | **P0.09 / P0.10** | D14 / D15 (NFC) | 完全未使用。テストパッドとして引き出し推奨 |

v1 からの改善点: 電池残量が BLE で見える／エンコーダも行も全て通常 GPIO
（NFC 開放の UICR 設定すら不要）／A/B をデータシートどおりに配線して
入れ替え不要にする／予備パッド 2 本確保。

基板設計時の注意: エンコーダの金属フレームは GND へ落とす。BAT パッドは
XIAO 内蔵の充電・分圧回路をそのまま使う（外部回路不要）。ファームは
`xileft_pad_v2.uf2` を使う（v1 の `.uf2` とはピン定義が違うので取り違え注意）。
