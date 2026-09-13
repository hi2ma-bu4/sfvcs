# sfvcs 基本設計書: リモート同期 Wire Protocol & フレーム構造 & エラーハンドリング
(`bd-network-wire-protocol.md`)

---

# 1. 概要と目的

本設計書は、クライアントとリモートサーバー間でのオブジェクト同期・交渉を行う Wire Protocol のバイナリメッセージ構造、能力交渉 (Capability Negotiation)、および明示的構造化エラーフレームの基本設計書である。

本書は `doc/02_specs/spec-network.md` の第1節, 第2節 (2.1, 2.2, 2.3) および `doc/01_architecture/sfvcs-design.md` のネットワーク通信仕様を完全網羅し、カプセル化された Network Protocol Codec Layer モジュールとして詳細を定義する。

---

# 2. カプセル化境界とモジュール構造

本モジュールは **Network Communication & Protocol Layer** に位置し、ソケットおよびトランスポートの直上でメッセージのパース・シリアライズを行う。

```
+-----------------------------------------------------------------------+
|                Network Sync Services (Fetch / Push / Clone)           |
+-----------------------------------------------------------------------+
                                   | Send / Receive Message Frames
                                   v
+-----------------------------------------------------------------------+
| bd-network-wire-protocol (Wire Protocol Codec Service)                |
|  - TypeScript: Transport Session & Framing Pipe Manager               |
|  - Rust/WASM Core: Binary Frame Serializer / Deserializer (zero-copy) |
+-----------------------------------------------------------------------+
                                   | Raw Byte Streams
                                   v
+-----------------------------------------------------------------------+
|        Transport Stream Layer (TCP / QUIC / WebSocket / SSH)          |
+-----------------------------------------------------------------------+
```

---

# 3. Wire Protocol メッセージ構造 & フレームレイアウト

すべての Wire Protocol 通信メッセージは以下のヘッダーフレーム構造を持つ。多重化通信（Multiplexing）に対応するため Channel ID を保持する。

```
+------------------------------------------------------------------------+
| Frame Type (uint8)                                                     |
+------------------------------------------------------------------------+
| Channel ID (uint16 BE)                                                 |
+------------------------------------------------------------------------+
| Payload Length (uint32 BE)                                             |
+------------------------------------------------------------------------+
| Payload Bytes (Payload Length バイト)                                  |
+------------------------------------------------------------------------+
```

### 3.1 Frame Type 一覧 (全18種完全一致)
| Type ID | 名称                         | 説明                                                      |
| ------- | ---------------------------- | --------------------------------------------------------- |
| `0x00`  | `MSG_HANDSHAKE_CAPABILITIES` | 初期ハンドシェイクおよび機能交渉 (Capability Negotiation) |
| `0x01`  | `MSG_REF_DISCOVERY`          | 参照 (Refs) 一覧の広告・要求                              |
| `0x02`  | `MSG_TREE_NEGOTIATE_REQ`     | 差分オブジェクト探索交渉リクエスト                        |
| `0x03`  | `MSG_TREE_NEGOTIATE_RESP`    | 差分オブジェクト探索交渉レスポンス                        |
| `0x04`  | `MSG_PACKFILE_DATA`          | パックデータ（オブジェクト群）ストリーム                  |
| `0x05`  | `MSG_REF_UPDATE_REQ`         | リモート Ref の CAS 更新リクエスト                        |
| `0x06`  | `MSG_REF_UPDATE_RESP`        | リモート Ref 更新結果応答                                 |
| `0x07`  | `MSG_LAZY_FETCH_REQ`         | Sparse/Shallow 向けオンデマンドオブジェクト要求           |
| `0x08`  | `MSG_LAZY_FETCH_RESP`        | オンデマンド要求オブジェクトデータ返答                    |
| `0x09`  | `MSG_LFS_POINTER_REQ`        | LFS 大容量オブジェクト取得リクエスト                      |
| `0x0A`  | `MSG_LFS_DATA`               | LFS チャンクドストリーミングデータ                        |
| `0x0B`  | `MSG_DYNAMIC_RANGE_REQ`      | オンデマンド部分ツリー範囲リクエスト                      |
| `0x0C`  | `MSG_DYNAMIC_RANGE_RESP`     | オンデマンド部分ツリー範囲レスポンス                      |
| `0x0D`  | `MSG_PREFETCH_BATCH_REQ`     | バッチプリフェッチ要求                                    |
| `0x0E`  | `MSG_PROGRESS`               | 進行状況テキスト ("Counting objects...", etc.)            |
| `0x0F`  | `MSG_ERROR`                  | 明示的構造化エラーメッセージ                              |
| `0x10`  | `MSG_SESSION_TOKEN`          | セッション再開トークン通知                                |
| `0x11`  | `MSG_RESUME_STREAM_REQ`      | QUIC / HTTP-3 ストリーム再開リクエスト                    |

---

# 4. 初期ハンドシェイク & 機能交渉 (Capability Negotiation)

接続確立直後、双方のクライアントおよびサーバーは `MSG_HANDSHAKE_CAPABILITIES` (`0x00`) フレームを交換し、互いにサポートするプロトコル機能をネゴシエーションする。

### 4.1 ペイロードレイアウト (`MSG_HANDSHAKE_CAPABILITIES`)
- `Protocol Version` (uint16 BE)
- `Supported Hash Algorithms Bitmask` (uint8):
  - Bit 0 (`0x01`): SHA-256
  - Bit 1 (`0x02`): BLAKE3
- `Compression Algorithms Bitmask` (uint8):
  - Bit 0 (`0x01`): Deflate/zlib
  - Bit 1 (`0x02`): Brotli
- `Capabilities Bitmask` (uint32 BE):
  - Bit 0 (`0x0001`): `CAP_THIN_PACK`
  - Bit 1 (`0x0002`): `CAP_LAZY_FETCH` (Sparse/Shallow)
  - Bit 2 (`0x0004`): `CAP_SIDEBAND_PROGRESS`

---

# 5. 明示的構造化エラーフレーム (`MSG_ERROR: 0x0F`)

ソケット切断や抽象メッセージではなく、型定義された構造化エラーを送信する。

```
+------------------------------------------------------------------------+
| Error Code (uint16 BE)                                                 |
+------------------------------------------------------------------------+
| Error Message Length (varint)                                          |
+------------------------------------------------------------------------+
| Error Message Bytes (UTF-8)                                            |
+------------------------------------------------------------------------+
```

- 主なエラーコード:
  - `0x0001`: `ERR_UNSUPPORTED_VERSION` (プロトコルバージョン不適合)
  - `0x0002`: `ERR_HASH_ALGO_MISMATCH` (ハッシュアルゴリズム不適合)
  - `0x0003`: `ERR_NON_FAST_FORWARD` (Push 時の非 Fast-Forward 更新拒否)
  - `0x0004`: `ERR_OBJECT_NOT_FOUND` (リクエストオブジェクト非存在)

---

# 6. セッション再開 & QUIC 0-RTT ストリーム復元モデル (`0x10`, `0x11`)

モデム断線や Wi-Fi/5G モバイル環境におけるネットワーク瞬断・切断時、途中まで転送したパックデータを破棄せず転送位置から高速再開する。

### 6.1 `MSG_SESSION_TOKEN` (`0x10`) & `MSG_RESUME_STREAM_REQ` (`0x11`)
- **`MSG_SESSION_TOKEN` (`0x10`)**: サーバーが転送開始時に 32 バイト暗号的ランダムトークン (有効期間: 24時間) を発行。
- **`MSG_RESUME_STREAM_REQ` (`0x11`)**:
  ```
  [Session_Token: 32 bytes][Last_Received_Byte_Offset: uint64 BE][Channel_ID: uint16 BE]
  ```

---

# 7. Rust / WASM Core & TypeScript インターフェース

```rust
pub struct WireFrameHeader {
    pub frame_type: u8,
    pub channel_id: u16,
    pub payload_length: u32,
}

pub fn parse_wire_frame_header(bytes: &[u8]) -> Result<WireFrameHeader, &'static str> {
    if bytes.len() < 7 {
        return Err("Header buffer too small");
    }
    Ok(WireFrameHeader {
        frame_type: bytes[0],
        channel_id: u16::from_be_bytes([bytes[1], bytes[2]]),
        payload_length: u32::from_be_bytes([bytes[3], bytes[4], bytes[5], bytes[6]]),
    })
}
```

```typescript
export interface WireFrameModel {
  type: number;
  channelId: number;
  payload: Uint8Array;
}

export interface WireProtocolCodecFacade {
  encodeFrame(frame: WireFrameModel): Uint8Array;
  decodeFrame(stream: Uint8Array): WireFrameModel;
}
```
