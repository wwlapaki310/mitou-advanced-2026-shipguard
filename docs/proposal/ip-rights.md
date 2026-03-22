# 知的財産権の権利情報

未踏アドバンスト提案フォーム「知的財産権の権利情報」記入用

---

## 1. 本プロジェクトで新たに創出する知的財産

本プロジェクトで開発するアプリケーション「小型船舶操縦士向け操縦補助アプリ」のソースコード・設計・モデルデータは、
規定に基づき提案者（秋田 智）に帰属する。

具体的に創出を予定しているもの：

- アプリケーション本体のソースコード（Flutter / Dart）
- 海事特化 YOLO11n Fine-tuning済みモデルの重みファイル
  （SMD・SeaShips・自前撮影データ等で追加学習したもの）
- 信号旗・形象物検知用学習データセット（自前収集・アノテーション分）
- 海事知識 RAG 用ドキュメントデータベース（独自編集・整備分）
- 安全確認行動モニタリングのロジック・アルゴリズム実装

---

## 2. 第三者が保有する知的財産権の利用

本プロジェクトでは以下のOSSおよびデータセットを利用する。
いずれも各ライセンスの条件の範囲内で利用する。

### 2-1. AIモデル・推論エンジン

| 名称 | 権利者 | ライセンス | 利用目的 | 備考 |
|---|---|---|---|---|
| YOLO11n | Ultralytics Inc. | **AGPL v3** | 物体検知モデルのベースとして Fine-tuning | AGPL v3 のため、派生物のソースコード開示義務あり。App Store 配布にあたっては Ultralytics 商用ライセンスの取得または開発モデルのみ利用（推論エンジン分離）を検討する。 |
| llama.cpp | Georgi Gerganov 他 | MIT | LLM 推論エンジン（Android JNI / iOS Swift Bindings） | 商用利用可 |
| Gemma 3 2B Instruct | Google LLC | Gemma Terms of Use | オフライン LLM の基盤モデル | 非商用・研究利用は無償。商用配布には Google の利用規約遵守が必要。再配布時はモデル重みの直接同梱ではなく初回ダウンロード方式で対応予定。 |
| multilingual-e5-small | Microsoft Corporation | MIT | RAG 用 Embedding モデル | 商用利用可 |

### 2-2. フレームワーク・ライブラリ

| 名称 | 権利者 | ライセンス | 利用目的 |
|---|---|---|---|
| Flutter SDK | Google LLC | BSD 3-Clause | アプリ開発フレームワーク |
| TensorFlow Lite | Google LLC | Apache 2.0 | Android 向けモデル推論ランタイム |
| Core ML | Apple Inc. | Apple Developer Program License | iOS 向けモデル推論ランタイム |
| sensors_plus | Flutter Community | BSD 2-Clause | IMU センサーデータ取得 |
| camera | Flutter Community | BSD 2-Clause | カメラフレーム取得 |
| flutter_tts | Davit Tchintcharauli | BSD 2-Clause | テキスト音声合成（TTS） |
| sqlite-vec | Stephen Marz | MIT / Apache 2.0 | モバイル向けベクトル検索 DB |
| drift (sqflite) | Simon Binder | MIT | SQLite ORM |
| ONNX Runtime Mobile | Microsoft Corporation | MIT | Embedding モデル推論 |
| go_router | Flutter Team | BSD 3-Clause | アプリ内ルーティング |
| riverpod | Remi Rousselet | MIT | 状態管理 |

### 2-3. 学習データセット

| 名称 | 提供元 | ライセンス | 利用目的 | 備考 |
|---|---|---|---|---|
| Singapore Maritime Dataset (SMD / SMD-Plus) | Dilip K. Prasad ほか | 学術・研究利用（非商用） | 船舶・浮遊物・落水者クラスの Fine-tuning | 商用配布アプリへのモデル組み込みにあたり、ライセンス条件の確認・必要に応じ権利者への問い合わせを実施する |
| SeaShips | Zhenwei Shao ほか | 学術・研究利用 | 船舶検知の Fine-tuning | 同上 |
| KOLOMVERSE | KRISO（韓国） | 要申請（研究利用） | ブイ・漁業設備クラスの Fine-tuning | データ利用申請を実施済みまたは実施予定 |

---

## 3. ライセンス上の主な課題と対応方針

### YOLO11n（AGPL v3）の商用配布問題

AGPL v3 は、ネットワーク越しにサービスを提供する場合も含めソースコード開示を要求するコピーレフトライセンスである。App Store / Google Play での配布時に問題となりうる。

対応方針：

**案A（推奨）：Ultralytics 商用ライセンスを取得する**
- Ultralytics は AGPL v3 の他に商用ライセンスを提供しており、App Store 配布に対応した条件での利用が可能。
- 費用は年間数万円〜（プランにより異なる）。委託費（手取り）から自己負担で対応。

**案B：AGPL v3 の条件を満たしてオープンソースで配布する**
- アプリ本体のソースコードを GitHub 等で公開することで AGPL v3 の条件を満たす。
- 知財帰属はイノベータにあるが、公開することで未踏の趣旨（オープンな技術発展）とも整合する。
- 本プロジェクトは当初よりリポジトリをパブリックに公開しており、この方針と整合しやすい。

**案C：YOLO11n の代替モデルを採用する**
- Apache 2.0 ライセンスの物体検知モデル（EfficientDet 等）へ切り替える。
- ただし YOLO11n と比較して推論速度・精度のトレードオフが生じる。

現時点では **案B（オープンソース配布）または案A（商用ライセンス取得）** を採用する方針とし、プロジェクト期間中に最終判断する。

### Gemma 3 の再配布

モデル重みをアプリに直接バンドルせず、初回起動時にユーザーのデバイスへダウンロードさせる方式を採用することで、Gemma Terms of Use における再配布の制約を回避する。

### 学習データセットの商用利用

SMD・SeaShips は学術利用を前提としたデータセットであるため、これらを用いて学習したモデルをアプリに組み込む商用配布については、必要に応じて権利者への確認・ライセンス交渉を実施する。なお、自前収集データ（信号旗・形象物・日本近海撮影）を中心に学習することで、SMD 等への依存度を低減する設計とする。

---

最終更新：2026-03-23
