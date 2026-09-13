# sfvcs 基本設計書: リモート同期 Wire Protocol & フレーム構造 & エラーハンドリング
(`bd-network-wire-protocol.md`)

---

# 1. 概要と目的

本設計書は、クライアントとリモートサーバー間でのオブジェクト同期・交渉を行う Wire Protocol のバイナリメッセージ構造、能力交渉 (Capability Negotiation)、および明示的構造化エラーフレームの基本設計書である。

本書は `doc/specs/spec-network.md` の第1節, 第2節 (2.1, 2.2, 2.3) および `doc/architecture/sfvcs-design.md` のネットワーク通信仕様を完全網羅し、カプセル化された Network Protocol Codec Layer モジュールとして詳細を定義する。

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

すべての Wire Protocol 通信メッセージは以下のヘッダーフレーム構造を持つ。

```
+------------------------------------------------------------------------+
| Magic: "SFPX" (4B) | Version (u16 LE) | Frame Type (u8) | Flags (u8)   |
| Payload Length (u32 LE) | Payload Data (Varint len) | BLAKE3 (32B)     |
+------------------------------------------------------------------------+
```

### 3.1 Frame Type 一覧
- `0x01` `MSG_HANDSHAKE_CAPABILITIES`: 初期機能交渉。
- `0x02` `MSG_WANT_OBJECTS`: クライアントが要求する CID 一覧。
- `0x03` `MSG_HAVE_OBJECTS`: クライアントが保持する CID 一覧。
- `0x04` `MSG_OBJECT_STREAM`: オブジェクトデータストリーム。
- `0x05` `MSG_ACK`: 完了受諾。
- `0x06` `MSG_NAK`: 失敗否定。
- `0x0D` `MSG_PREFETCH_BATCH_REQ`: バッチプリフェッチ要求。
- `0x0F` `MSG_ERROR`: 明示的構造化エラー。

---

# 4. 初期ハンドシェイク & 機能交渉 (Capability Negotiation)

接続確立直後、双方のクライアントおよびサーバーは `MSG_HANDSHAKE_CAPABILITIES` フレームを交換し、互いにサポートするプロトコル機能をネゴシエーションする。

### 4.1 ペイロードレイアウト (`MSG_HANDSHAKE_CAPABILITIES`)
- `protocol_version` (u16 LE)
- `hash_algo` (u8): `0x02` (BLAKE3), `0x01` (SHA-256)
- `compression_algo` (u8): `0x01` (Zstandard), `0x02` (Deflate)
- `capabilities_mask` (u64 LE):
  - `0x0001`: `CAP_SPARSE_OBJECTS` (部分ツリー取得)
  - `0x0002`: `CAP_SHALLOW_CLONE` (浅い履歴 clone)
  - `0x0004`: `CAP_LFS_STREAMING` (LFS オフロード)
  - `0x0008`: `CAP_QUIC_RESUME` (セッション再開)
  - `0x0010`: `CAP_ENCRYPTED_REPO` (ゼロ知識暗号化ストレージ)
- `client_agent_string` (UTF-8 NFC)

---

# 5. 明示的構造化エラーフレーム (`MSG_ERROR: 0x0F`)

ソケット切断や抽象メッセージではなく、型定義された構造化エラーを送信する。

```
+------------------------------------------------------------------------+
| Error Code (u32 LE) | Error Category (u8) | Message Length (Varint)    |
| UTF-8 Error Message | JSON Details Payload                             |
+------------------------------------------------------------------------+
```

---

# 6. Rust / WASM Core & TypeScript インターフェース

```rust
pub struct WireFrameHeader {
    pub magic: [u8; 4],
    pub version: u16,
    pub frame_type: u8,
    pub flags: u8,
    pub payload_length: u32,
}

pub fn parse_wire_frame_header(bytes: &[u8]) -> Result<WireFrameHeader, &'static str> {
    if bytes.len() < 12 || &bytes[0..4] != b"SFPX" {
        return Err("Invalid SFPX Header");
    }
    Ok(WireFrameHeader {
        magic: [bytes[0], bytes[1], bytes[2], bytes[3]],
        version: u16::from_le_bytes([bytes[4], bytes[5]]),
        frame_type: bytes[6],
        flags: bytes[7],
        payload_length: u32::from_le_bytes([bytes[8], bytes[9], bytes[10], bytes[11]]),
    })
}
```

```typescript
export interface WireFrameModel {
  type: number;
  flags: number;
  payload: Uint8Array;
}

export interface WireProtocolCodecFacade {
  encodeFrame(frame: WireFrameModel): Uint8Array;
  decodeFrame(stream: Uint8Array): WireFrameModel;
}
```
