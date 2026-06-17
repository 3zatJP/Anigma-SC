# Anigma-SC (System-Core)


> **Anigma** のコアプロトコルを汎用化した、時間同期型・揮発性鍵・オフライン暗号化トランスポートエンジン。

-----

## Overview

Anigma-SC は [Anigma](https://github.com/3zatJP/Anigma-kt) の根幹を成す3つの設計原則を、特定のアプリケーション（QRコード転送）から切り離して再定義したものです。

|原則                     |概要                              |
|-----------------------|--------------------------------|
|**Ephemeral Key**      |鍵はメモリ上にのみ存在し、用途完了と同時にゼロ化される     |
|**Sync-Derived Keying**|共有シークレットと時間/イベント同期トークンから鍵を導出する  |
|**Transport Agnostic** |搬送路（QR・NFC・ファイル・音響等）をプロトコルから分離する|

-----

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     Anigma-SC                           │
│                                                         │
│  ┌─────────────────┐     ┌──────────────────────────┐  │
│  │  KDF Engine     │────▶│  Crypto Context          │  │
│  │                 │     │  (Ephemeral)             │  │
│  │  secret         │     │                          │  │
│  │    +            │     │  AES-256-GCM             │  │
│  │  sync_token     │     │  生存期間: 1 operation   │  │
│  │                 │     │  保存: なし              │  │
│  └─────────────────┘     └──────────┬───────────────┘  │
│                                     │                   │
│                          ┌──────────▼───────────────┐  │
│                          │  Transport Layer         │  │
│                          │  (実装側が定義する)       │  │
│                          │                          │  │
│                          │  QR / NFC / File /       │  │
│                          │  BLE / Print / Voice ... │  │
│                          └──────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

-----

## Core Specification

### Layer 1 — KDF Engine

共有シークレットと同期トークンを組み合わせて、揮発性の暗号鍵を生成します。

```
Key-main = Argon2( SHA3-256( secret || sync_token ) )
```

|パラメータ       |SC仕様           |実装側の自由度           |
|------------|---------------|------------------|
|`secret`    |任意のバイト列        |PIN・パスフレーズ・生体トークン等|
|`sync_token`|時間/カウンター/イベント由来|TOTP・HOTP・チャレンジ等  |
|ハッシュ関数      |SHA3-256（推奨）   |差し替え可（要セキュリティ評価）  |
|KDF         |Argon2（推奨）     |差し替え可（要セキュリティ評価）  |
|出力          |256bit         |固定                |

### Layer 2 — Ephemeral Crypto Context

Anigma-SC の最重要設計思想。鍵を「使い捨て文脈」として扱い、永続化を原則禁止とします。

**ライフサイクル:**

```
sync_token 更新イベント
        │
        ▼
  既存鍵をゼロ化
        │
        ▼
  新しい鍵を生成（Layer 1）
        │
        ▼
  暗号化 or 復号化（1回のみ）
        │
        ▼
  ┌─────────────────────┐
  │ 成功 / 失敗 / タイムアウト │
  └──────────┬──────────┘
             │
             ▼
        鍵をゼロ化・破棄
```

**破棄条件（いずれかを満たした時点）:**

- sync_token の更新
- 暗号化/復号化の完了（成功・失敗問わず）
- タイムアウト（実装側が定義）

**禁止事項:**

- 鍵のディスク保存
- 鍵のログ出力
- 鍵の複数用途での再利用

### Layer 3 — Transport Layer (実装側が定義)

Anigma-SC はペイロードの搬送路を規定しません。実装側が以下のインターフェースを満たす形で自由に定義します。

**ペイロード構造:**

```
SC-Payload = Base64( AES-256-GCM出力 )

AES-256-GCM出力:
  [IV (96bit)] [暗号文] [認証タグ (128bit)]
```

**搬送路の実装例:**

|実装名            |搬送路  |想定用途            |
|---------------|-----|----------------|
|Anigma         |QRコード|スマートフォン間のオフライン移行|
|Anigma-SC/File |ファイル |暗号化ファイルの共有      |
|Anigma-SC/NFC  |NFC  |近距離タップ転送        |
|Anigma-SC/Print|印刷紙  |物理的なオフライン転送     |
|Anigma-SC/Voice|音響符号化|ネットワーク非依存の音声転送  |

-----

## Interface Contract

Anigma-SC に準拠する実装が満たすべき契約です。

```
interface AnigmaSCTransport {
  // ペイロードを生成して搬送路に載せる
  send(payload: Base64String): void

  // 搬送路からペイロードを受け取る
  receive(): Base64String
}

interface AnigmaSCSync {
  // 現在の同期トークンを返す
  currentToken(): Bytes

  // トークン更新イベントを購読する
  onUpdate(callback: () => void): void
}
```

実装側はこの2つのインターフェースを満たすことで、KDF Engine および Ephemeral Crypto Context を変更なく利用できます。

-----

## Security Properties

| 性質         | 実現方法                                 |
| ---------- | ------------------------------------ |
| 前方秘匿性      | sync_token ごとに鍵を再生成、過去の鍵は復元不可        |
| リプレイ攻撃耐性   | sync_token の時間/カウンター性により古いペイロードは復号不可 |
| 中間者攻撃耐性    | 搬送路を物理/近距離に限定することで達成（実装側の責務）         |
| 鍵漏洩リスクの最小化 | メモリオンリー・即時ゼロ化ポリシー                    |
| 完全性・真正性保証  | AES-GCM の認証タグにより改ざん検知                |

-----

## Relationship to Anigma

```
Anigma
  └── Anigma-SC (System-Core)   ← このリポジトリ
        ├── KDF Engine
        ├── Ephemeral Crypto Context
        └── Transport Interface
              └── QRコード実装   ← Anigma 本体
```

Anigma は Anigma-SC の Transport Layer を QRコードで実装したリファレンス実装です。

-----

## Notes

- 本仕様に準拠した実装においても、セキュリティ監査を受けることを強く推奨します。
- Argon2 パラメータ（memory, iterations, parallelism）は実装環境に応じて適切に設定してください。
- sync_token の同期精度は実装側の責務です。時刻ベースの場合は NTP 等による同期を推奨します。

-----

## Versioning

|バージョン|日付        |内容                                 |
|-----|----------|-----------------------------------|
|0.1.0|2026-06-16|Anigma v1.0.1 より System-Core を抽出・初版|

-----

## Related

- [Anigma](https://github.com/3zatJP/Anigma-kt) — QRコードベースのリファレンス実装
- RFC 6238 — TOTP: Time-Based One-Time Password Algorithm
- NIST SP 800-38D — Recommendation for Block Cipher Modes of Operation (GCM)