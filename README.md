# P2P Location Bond — AR Co-location 設計書

## 1. 概要

本システムは、2台のモバイル端末間でシグナリングサーバーを介さずにWebRTC DataChannelを確立し、互いのGPS位置情報をリアルタイムに交換して、AR空間上に相手の方向・距離を可視化するアプリケーションである。

- **対象端末**: WebRTC / Geolocation API / WebGL(A-Frame)対応のモバイルブラウザ
- **通信方式**: WebRTC DataChannel(P2P)、シグナリングはSDPの手動コピー&ペーストによる帯域外交換
- **表示方式**: AR.js + A-Frameによるカメラ映像への重畳表示

## 2. 目的・スコープ

| 項目 | 内容 |
|---|---|
| 目的 | サーバーを介さず、2者間の相対距離をARで直感的に把握できるようにする |
| スコープ内 | GPS取得、WebRTC手動シグナリング、位置同期、距離計算、AR表示 |
| スコープ外 | シグナリングサーバーの自動化、3者以上のメッシュ通信、認証・暗号化の追加実装(WebRTC標準のDTLSに依存) |

## 3. システム構成

```
[端末A]                                   [端末B]
 GPS ──▶ State.localPos                    GPS ──▶ State.localPos
   │                                          │
   ├─ WebRTC DataChannel ◀───────────────────┤
   │      (SDP手動交換 / STUN経由ICE)         │
   │                                          │
   ▼                                          ▼
 State.remotePos/filteredPos           State.remotePos/filteredPos
   │                                          │
   ▼                                          ▼
 距離計算(Haversine) ─▶ AR描画(bond-system) ◀─ 距離計算(Haversine)
```

## 4. 主要コンポーネント

### 4.1 State(グローバル状態)

| フィールド | 型 | 説明 |
|---|---|---|
| `localPos` | `{lat, lng}` | 自端末のGPS座標 |
| `localAccuracy` | number | GPS精度(m) |
| `remotePos` | `{lat, lng}` | 相手から直近で受信した生の座標 |
| `filteredPos` | `{lat, lng}` | 平滑化フィルター後の相手座標(AR表示に使用) |
| `distance` | number \| null | 自端末と`filteredPos`間の距離(m) |
| `lastSent` / `lastReceived` | timestamp \| null | 送受信の最終時刻。通信途絶検知に使用 |
| `pc` | `RTCPeerConnection` | WebRTC接続オブジェクト |
| `dc` | `RTCDataChannel` | 位置情報送受信用チャネル |

### 4.2 位置フィルタリング

相手から受信した座標をそのまま使わず、指数移動平均(EMA)で平滑化する。

```
filteredPos = filteredPos * (1 - SMOOTHING) + remotePos * SMOOTHING
```

- `SMOOTHING = 0.3`固定値。GPSのジッターを軽減しつつ、追従遅延を抑える。
- 初回受信時は`filteredPos`が存在しないため、`remotePos`をそのまま初期値とする。

### 4.3 距離計算

Haversine公式による球面距離計算(`calcDistance`)。地球半径`R = 6371000m`を使用。

### 4.4 WebRTCシグナリング(手動SDP交換)

シグナリングサーバーを持たないため、SDPの受け渡しは利用者がテキストをコピー&ペーストして行う。

**Offer側(発信者)の手順**
1. `createOfferBtn` 押下 → `RTCPeerConnection`生成、DataChannel(`bond`)作成
2. `createOffer()` → `setLocalDescription()`
3. ICE候補収集完了(または5秒タイムアウト)を待機
4. 完成したSDPをテキストエリアに出力 → 相手に送付

**Answer側(応答者)の手順**
1. 受け取ったOffer SDPを入力欄に貼り付け
2. `acceptRemoteBtn` 押下 → `RTCPeerConnection`生成、`setRemoteDescription(offer)`
3. `createAnswer()` → `setLocalDescription()`
4. ICE候補収集完了(または5秒タイムアウト)を待機
5. 完成したAnswer SDPを出力 → 発信者に送付

**発信者側の最終手順**
1. 受け取ったAnswer SDPを入力欄に貼り付け
2. `setSdpBtn` 押下 → `setRemoteDescription(answer)`
3. 接続確立(`connectionState: connected`)後、DataChannelがopenになり位置送信ループが起動

### 4.5 ICE候補収集(`waitIceGatheringComplete`)

`iceGatheringState`が`complete`になるのを待つが、対称NAT環境などで`complete`に至らないケースがあるため、**5秒のタイムアウト**を設けている。タイムアウト時はその時点までに集まった候補でSDPを確定する。

### 4.6 位置送信ループ

DataChannelが`open`になった時点で2秒間隔の`setInterval`を開始し、`{type: 'pos', pos, acc}`形式のJSONを送信する。DataChannelが閉じた際はループを確実にクリアする。

### 4.7 通信途絶検知

`lastReceived`を1秒ごとのUIポーリングで参照し、**10秒(`STALE_THRESHOLD_MS`)以上受信がない場合**、距離表示と接続線をグレーアウトして「情報が古い可能性がある」ことを視覚的に示す。

### 4.8 AR描画(`bond-system`コンポーネント)

A-Frameの`tick`ハンドラで毎フレーム実行。

1. 自端末・相手の座標が両方揃っていなければ何もしない(防御処理)
2. 距離をテキスト表示(1000m未満は m、以上は km)
3. `filteredPos`を`gps-entity-place`に設定し、相手位置にマーカー(GLTFモデル)を配置
4. カメラとマーカーのワールド座標を`updateMatrixWorld(true)`で強制同期後に取得し、両者を結ぶ接続線(`line`)を再描画
5. マーカー未生成時のマトリクス未定義例外は`try-catch`で握りつぶし、当該フレームの描画をスキップ

## 5. 設定値一覧

| 定数 | 値 | 説明 |
|---|---|---|
| `SMOOTHING` | 0.3 | 位置フィルターのEMA係数 |
| `ICE_GATHERING_TIMEOUT_MS` | 5000 | ICE候補収集の打ち切り時間 |
| `STALE_THRESHOLD_MS` | 10000 | 通信途絶とみなす経過時間 |
| STUNサーバー | `stun.l.google.com:19302` | Google公開STUN。TURNサーバーは未設定 |
| 位置送信間隔 | 2000ms | DataChannel経由の位置送信周期 |

## 6. 既知の制限事項

- **TURNサーバー未設定**: 対称NAT配下同士では直接接続が確立できない可能性がある。安定運用にはTURNサーバーの追加が必要。
- **シグナリングの手動性**: SDPのコピー&ペーストはUXとして煩雑であり、誤操作(貼り付け忘れ・順序間違い)によって接続失敗しうる。
- **再接続導線なし**: 接続が切断された場合、明示的な再接続フローがなく、事実上ページのリロードが必要。
- **認証機構なし**: 接続相手の正当性を検証する仕組みがないため、Offer/Answerを知っていれば誰でも接続しうる(WebRTCのDTLS暗号化により通信内容の盗聴は防げるが、なりすまし対策ではない)。
- **GPS精度依存**: 屋内や高層ビル街ではGPS精度が大きく劣化し、距離表示・AR表示の信頼性が下がる。

## 7. 今後の課題

- シグナリングサーバー(WebSocket等)の導入による自動化
- TURNサーバー追加によるNAT越え成功率の向上
- 再接続・複数ペア(3者以上)対応の検討
- 通信途絶からの自動再送・再同期ロジック

## 8. 変更履歴

| バージョン | 内容 |
|---|---|
| v1 | 初版。TDGL演出・確率的ジッター・パルスアニメーションを含む実装 |
| v2 | 装飾要素(TDGL、パルス、確率的ジッター、色相アニメーション)を削除。以下のバグを修正: `filteredPos`参照の`ReferenceError`、ICE収集の無限待機、`raw.githack.com`依存の解消。通信途絶検知(`STALE_THRESHOLD_MS`)を追加 |
