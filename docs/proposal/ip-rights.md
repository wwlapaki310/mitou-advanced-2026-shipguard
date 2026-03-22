# 知的財産権の権利情報

未踏アドバンスト提案フォーム「知的財産権の権利情報」記入用

---

## 1. 本プロジェクトで新たに創出する知的財産

本プロジェクトで開発するアプリケーションのソースコード・設計・モデルデータは、規定に基づき提案者（秋田 智）に帰属する。

創出を予定しているもの：

- アプリケーション本体のソースコード（Flutter / Dart）
- 海事特化 Fine-tuning 済み物体検知モデルの重みファイル（自前収集・アノテーション済みデータによる追加学習分）
- 信号旗・形象物検知用学習データセット（自前収集・アノテーション分）
- 海事知識 RAG 用ドキュメントデータベース（独自編集・整備分）
- 安全確認行動モニタリングのロジック・アルゴリズム実装

なお、アプリケーション本体のソースコードはオープンソースとして公開する方針とする。

---

## 2. 第三者が保有する知的財産権の利用

本プロジェクトでは以下の OSS・データセット・モデルを利用する予定である。いずれも各ライセンスの条件の範囲内で利用する。

### 2-1. AIモデル・推論エンジン

| 名称 | 権利者 | ライセンス | 利用目的 |
|---|---|---|---|
| オブジェクト検知モデル（YOLO 系またはそれに準ずる軽量モデル） | 採用モデルにより異なる | 採用モデルのライセンスに従う | 物体検知の Fine-tuning ベース。ライセンス条件を確認のうえ、アプリのオープンソース公開方針と整合するものを選定する |
| llama.cpp | Georgi Gerganov 他 | MIT | LLM 推論エンジン（Android JNI / iOS Swift Bindings） |
| LLM 基盤モデル（Gemma 3 または同等の軽量モデル） | Google LLC 他 | 採用モデルの利用規約に従う | オフライン LLM。モデル重みは初回ダウンロード方式とし、再配布制約を回避する |
| multilingual-e5-small（または同等の Embedding モデル） | Microsoft Corporation 他 | MIT 相当 | RAG 用 Embedding |

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
| drift（sqflite） | Simon Binder | MIT | SQLite ORM |
| ONNX Runtime Mobile | Microsoft Corporation | MIT | Embedding モデル推論 |
| go_router | Flutter Team | BSD 3-Clause | アプリ内ルーティング |
| riverpod | Remi Rousselet | MIT | 状態管理 |

### 2-3. 学習データセット

| 名称 | 提供元 | ライセンス | 利用目的 | 備考 |
|---|---|---|---|---|
| Singapore Maritime Dataset（SMD / SMD-Plus） | Dilip K. Prasad ほか | 学術・研究利用 | 船舶・浮遊物・落水者クラスの Fine-tuning | 商用配布時のライセンス条件を確認のうえ利用する |
| SeaShips | Zhenwei Shao ほか | 学術・研究利用 | 船舶検知の Fine-tuning | 同上 |
| KOLOMVERSE | KRISO（韓国） | 要申請（研究利用） | ブイ・漁業設備クラスの Fine-tuning | データ利用申請を実施予定 |

---

## 3. ライセンス方針

本プロジェクトで創出するアプリケーションのソースコードはオープンソースとして公開する。利用するライブラリ・モデルの選定にあたっては、このオープンソース公開方針と整合するライセンス（MIT / Apache 2.0 / BSD 等）を優先する。

学習データセットについては、自前収集データ（信号旗・形象物・日本近海撮影）を中心に構成することで、学術利用限定データセットへの依存度を低減する設計とする。

---

最終更新：2026-03-23
