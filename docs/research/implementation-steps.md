# 技術実装ステップ

## アーキテクチャ全体像

```
┌─────────────────────────────────────────────────────┐
│                    ShipGuard App                     │
│  (Flutter or React Native — iOS / Android 共通)      │
│                                                      │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ │
│  │  ① Detection │ │   ② IMU      │ │   ③ LLM      │ │
│  │  Module      │ │   Module     │ │   Module     │ │
│  │              │ │              │ │              │ │
│  │ Camera frame │ │ Gyro+Accel   │ │ Gemma/Qwen   │ │
│  │ → YOLO推論   │ │ → Roll/Pitch │ │ 量子化モデル  │ │
│  │ → BBox表示   │ │ → 危険度スコア│ │ + 海事RAG    │ │
│  └──────────────┘ └──────────────┘ └──────────────┘ │
│              ↓              ↓              ↓          │
│  ┌─────────────────────────────────────────────────┐ │
│  │                   HUD レイヤー                   │ │
│  │  カメラビュー上にBBox・ロール角・警告を重畳表示    │ │
│  └─────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

---

## モジュール① 高速外界検知

### 課題
- 海上は背景が単調（空・水面）だが、波・白波・光反射でノイズが多い
- 検知対象：他船舶、浮遊物（ブイ・流木・漁具）、岸壁・防波堤、泳者・落水者
- スマホCPU/NPU上で **15 FPS以上** を目指す（VLMでは不可能）

### 実装ステップ

**Step 1: データセット収集・整備**
- 公開データセット活用
  - Singapore Maritime Dataset (SMD) — 海上監視映像、船舶・人・浮遊物ラベル付き
  - SeaShips Dataset — 中国沿岸、7クラス、6000+画像
  - MARVEL Dataset — マルチモーダル海上データ
- 自前データ収集：スマホで撮影した実海域映像（横浜港・東京湾等）をアノテーション

**Step 2: モデル選定・学習**
```
候補モデル（FPS優先順）:
  YOLOv8n  — 3.2M params、最軽量、スマホ向き
  YOLOv8s  — 11.2M params、精度とのバランス型
  RT-DETR-small — Transformerベース、精度高め

学習環境:
  Google Colab Pro / ローカルGPU
  Fine-tuning: 海上特化クラスでYOLOv8nをSMD+自前データで再学習
```

**Step 3: モバイル変換・最適化**
```
Android: TFLite (FP16 / INT8量子化) or ONNX Runtime Mobile
iOS:     CoreML (ANE活用で高速化)

変換例 (YOLO → TFLite):
  yolo export model=yolov8n.pt format=tflite int8=True
  
目標: iPhone 14以降 / Snapdragon 8 Gen 2以降で 20+ FPS
```

**Step 4: 距離推定**
- 単眼カメラで距離を推定（ステレオなし）
- アプローチ：
  - バウンディングボックス高さから推定（クラス別既知サイズを利用）
  - MiDaS / Depth Anything v2 の軽量版（TFLite）併用を検討

**Step 5: HUD表示**
```flutter
// Flutter での Camera + ML Kit / TFLite 推論例（概念）
CameraPreview(controller)
  + CustomPaint で BBox・距離ラベルを重畳
  + 危険エリア（画面中央±X度）内に入ったら警告色に変更
```

---

## モジュール② 姿勢・危険度モニタリング

### 課題
- スマホのMEMS IMUは精度が低い（ジャイロドリフト あり）
- 船の動きとスマホの取付角度の分離が必要
- 波による高周波振動と「実際の船体傾斜」を区別する必要がある

### 実装ステップ

**Step 1: センサー取得**
```dart
// Flutter: sensors_plus パッケージ
import 'package:sensors_plus/sensors_plus.dart';

gyroscopeEvents.listen((event) { /* ωx, ωy, ωz */ });
accelerometerEvents.listen((event) { /* ax, ay, az */ });
// サンプリング: 50〜100 Hz
```

**Step 2: 姿勢角推定（Complementary Filter）**
```
Complementary Filter（補完フィルタ）:
  加速度から求めたロール/ピッチ角（低周波は正確）と
  ジャイロ積分から求めた角度変化（高周波は正確）を
  重み付き合成して安定した姿勢角を得る

  roll_est = α × (roll_prev + ωx × dt) + (1-α) × roll_accel
  α = 0.96 〜 0.98 が一般的

より精度が必要な場合は Madgwick Filter を採用
```

**Step 3: 取付角補正**
- アプリ起動時に「キャリブレーション」ボタンで現在の静止状態をゼロ点登録
- 以降の角度変化を相対値として使用

**Step 4: 転覆危険度スコアリング**
```
危険度モデル（ルールベース v1）:
  ロール角 < 15°  → 緑（安全）
  ロール角 15〜30° → 黄（注意）
  ロール角 > 30°  → 赤（危険）
  横加速度 > 2G   → 即時警告（急旋回・衝突衝撃）
  角速度変化率（ジャーク）が閾値超過 → 転覆予兆警告

将来: IMUログデータを蓄積 → 機械学習で危険度モデルを改善
```

**Step 5: HUD表示（人工水平儀風）**
- 航空計器の「姿勢指示器（ADI）」のUIをカメラビューに重畳
- 数値＋視覚的な傾き表現でロール・ピッチ角を直感表示

---

## モジュール③ ローカルLLM操船相談

### 課題
- 海上は圏外が多い → オフライン必須
- 量子化モデルはスマホストレージ（1〜3GB）を消費
- 海事ドメイン知識（法規・気象・操船）をどう注入するか

### 実装ステップ

**Step 1: ベースモデル選定**
```
候補（2025〜2026年時点）:
  Gemma 3 2B IT (Q4_K_M)  — Googleが提供、日本語対応、〜1.5GB
  Qwen2.5 1.5B Instruct   — アリババ、日本語・中国語強、〜1GB
  Phi-3.5 Mini            — Microsoft、英語特化だが軽量

推論エンジン:
  Android: llama.cpp (JNI経由) or MediaPipe LLM Inference API
  iOS:     llama.cpp (Swift binding) or Apple MLX
```

**Step 2: 海事知識ベース構築（RAG用）**
```
収録コンテンツ:
  - 海上衝突予防法（72COLREGS）日本語要約
  - 小型船舶操縦士試験テキスト（筆記問題集）
  - 気象・海象の基礎知識（風力階級、波高判断）
  - 緊急時対応手順（人身落水、機関故障、浸水）
  - よく聞かれるQ&A集（免許スクール監修を検討）

形式:
  Markdown → テキスト分割 → Embedding → ローカルベクトルDB
  (faiss-lite or sqlite-vss for mobile)
```

**Step 3: RAG + LLM推論パイプライン**
```
ユーザー入力
  → Embeddingで関連チャンク検索（ローカルVectorDB）
  → プロンプト組み立て: [コンテキスト] + [質問]
  → 量子化LLMで推論（ストリーミング出力）
  → テキスト表示

レイテンシ目安（Snapdragon 8 Gen 3 / A17 Pro）:
  First token: 〜1秒
  生成速度: 10〜20 tokens/sec
```

**Step 4: 安全対策**
- 「これは参考情報です。緊急時は海上保安庁118番に連絡してください」を必ず付記
- 「操縦中はLLMを使わないでください」のUX設計（停泊中・待機中のみ推奨表示）

---

## 開発環境・技術スタック（案）

| 層 | 技術選定 | 理由 |
|---|---|---|
| **フレームワーク** | Flutter | iOS/Android統一、カメラ・センサー・UI統合が容易 |
| **言語** | Dart + C/C++（FFI） | LLM推論はllama.cppをFFI経由で呼び出し |
| **物体検知** | YOLOv8n → TFLite/CoreML | FPS最優先、エコシステムが充実 |
| **IMU処理** | sensors_plus + Dart実装 | リアルタイム50Hz処理に十分 |
| **LLM推論** | llama.cpp + MediaPipe | クロスプラットフォーム対応 |
| **ベクトルDB** | sqlite-vss (RAG用) | 軽量、モバイル実績あり |
| **CI/CD** | GitHub Actions | テスト自動化・APKビルド |

---

## マイルストーン別 実装タスク

```
M1 (〜2ヶ月): 基盤構築
  - Flutter プロジェクト初期化
  - カメラプレビュー表示
  - IMUデータ取得・ログ保存
  - YOLOv8nのTFLite変換検証（PCで動作確認）

M2 (〜4ヶ月): ①②モジュール完成
  - YOLOv8n on-device推論 → BBoxリアルタイム表示
  - Complementary Filter実装 → Roll/Pitch HUD表示
  - 海上動画での検知精度検証（FPS・mAP計測）
  - 転覆危険度スコアのルールベース実装

M3 (〜6ヶ月): ③LLMモジュール統合
  - llama.cpp Flutter FFI組み込み
  - 海事知識ベース構築・RAGパイプライン実装
  - 3モジュール統合UI

M4 (〜8ヶ月): UX改善・実海域テスト
  - 免許スクールでユーザーテスト（n=10〜20）
  - バッテリー消費・熱問題の対策
  - β版TestFlight/Play Console配布

M5 (〜12ヶ月): 公開・事業化検討
  - App Store / Google Play 公開
  - フィードバック収集・改善
  - B2B展開（マリーナ・スクール向け）検討
```

---

## 技術的リスクと対策

| リスク | 内容 | 対策 |
|---|---|---|
| FPS不足 | 検知が遅くて実用に耐えない | INT8量子化・入力解像度削減・フレームスキップ |
| IMUドリフト | ジャイロ積分誤差の蓄積 | Complementary Filter + 定期キャリブレーション |
| 海上データ不足 | 学習データが少なく精度が低い | SMD+SeaShips+自前収集の組み合わせ |
| LLMサイズ | 3GB超モデルは配布困難 | Q4量子化（1〜1.5GB）でアプリ内課金方式でDL |
| 誤警告 | 波・光反射を障害物と誤検知 | 海上特化ファインチューニング・閾値調整UI |
| 法的責任 | 安全支援ツールとして過信される | 「補助ツール」の明示・免責事項・UX設計 |
