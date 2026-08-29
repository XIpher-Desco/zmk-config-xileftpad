# zmk-config-xileftpad

左手用キーパッド **xileft pad**（31キー + ロータリーエンコーダ 3個）の ZMK ファームウェア設定。

QMK/Vial 版 `xipher/xileft_pad_v3` からの移行。コントローラは RP2040-Zero から
**Seeed XIAO nRF52840 Plus** に変更、無線（BLE）対応。

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
「xileft pad」を追加。つながらないときは該当プロファイルで `BT_CLR` して
デバイス側の登録も削除し、やり直すのが確実。

---

## ハードウェア上の注意

このキーボードは XIAO nRF52840 Plus の GPIO を 18 本使い切っている
（COL 6 + ROW 6 + ROT 6）。overlay には以下の対処が入っている。

| 対処 | 理由 |
| --- | --- |
| `&uicr { nfct-pins-as-gpios; }` | ROT1/ROT2 が NFC 兼用ピン（P0.09 / P0.10）に載っているため |
| `&vbatt { status = "disabled"; }` | ROT3 が電池電圧読み取りピン（P0.31）を占有しているため |
| `&xiao_i2c` / `&xiao_spi` を disabled | 既定の pinctrl が COL4/5・ROW1-3 と重なるため（将来の衝突予防） |

`nfct-pins-as-gpios` は UICR に一度だけ書き込まれる。**元に戻すにはチップの
全消去が必要**なので注意（実用上の問題はない）。

電池は接続して動作するが、`vbatt` を無効化しているため**残量レポートは出ない**。
将来どうしても残量表示が欲しくなった場合は、ROT3 を D16 から未使用の D19 へ
ジャンパで移すリワークで解決できる。

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

基板の `LED` ネット（D6 / P1.11）は未使用。

---

## 既知の注意点

- `CONFIG_ZMK_POINTING`（マウスボタン / ホイール）の有効・無効を後から
  切り替えると HID ディスクリプタが変わるため、**BLE 接続済みのデバイスでは
  マウス系が効かなくなることがある**。その場合は BT レイヤの `BT_CLR` と
  デバイス側の登録削除で一度ペアリングを消し、繋ぎ直すこと。
- 電池は接続して動作するが、ROT3 が電池電圧読み取りピン（P0.31）を
  占有しているため**残量レポートは出ない**（設計上の割り切り）。
