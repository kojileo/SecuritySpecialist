# IPsec入門

## 目次
1. [IPsecとは](#ipsecとは)
2. [IPsecの基本概念](#ipsecの基本概念)
3. [IPsecのアーキテクチャ](#ipsecのアーキテクチャ)
4. [IPsecプロトコル](#ipsecプロトコル)
5. [鍵管理システム](#鍵管理システム)
6. [IPsecの動作モード](#ipsecの動作モード)
7. [VPN実装](#vpn実装)
8. [設定と運用](#設定と運用)
9. [セキュリティ考慮事項](#セキュリティ考慮事項)
10. [トラブルシューティング](#トラブルシューティング)
11. [実際のケーススタディ](#実際のケーススタディ)
12. [まとめ](#まとめ)

---

## IPsecとは

**IPsec（Internet Protocol Security）** は、IPレイヤーでネットワーク通信を暗号化・認証するセキュリティプロトコル群です。インターネット上での安全な通信を実現するための標準的な仕組みとして広く使用されています。

### 基本概念
- **ネットワークレイヤーセキュリティ**: OSI参照モデルの第3層（ネットワーク層）でのセキュリティ
- **透過的な保護**: アプリケーションの変更なしにセキュリティを提供
- **エンドツーエンド暗号化**: 送信元から宛先まで一貫したセキュリティ
- **相互認証**: 通信相手の身元確認

### IPsecの目的
1. **機密性（Confidentiality）**: 盗聴からの保護
2. **完全性（Integrity）**: データ改ざんの検出
3. **認証（Authentication）**: 通信相手の身元確認
4. **リプレイ攻撃防止**: 過去のパケットの再送攻撃を防ぐ

### 使用場面
- **VPN（Virtual Private Network）**: 拠点間接続、リモートアクセス
- **セキュアな通信**: 機密データの送受信
- **ネットワーク間接続**: 信頼できないネットワーク経由の通信
- **クラウド接続**: オンプレミスとクラウド間の安全な接続

---

## IPsecの基本概念

### セキュリティアソシエーション（SA）
```
SA（Security Association）= IPsecにおける通信の約束事

SA情報:
├── SPI（Security Parameter Index）: SA識別子
├── 宛先IPアドレス
├── セキュリティプロトコル（AH/ESP）
├── 暗号化アルゴリズム
├── 認証アルゴリズム
├── 鍵情報
└── SA有効期限
```

### SA Database（SAD）
```python
# SA Database の概念例
class SecurityAssociation:
    def __init__(self):
        self.spi = None              # Security Parameter Index
        self.destination_ip = None    # 宛先IPアドレス
        self.protocol = None         # AH or ESP
        self.encryption_alg = None   # 暗号化アルゴリズム
        self.auth_alg = None         # 認証アルゴリズム
        self.encryption_key = None   # 暗号化鍵
        self.auth_key = None         # 認証鍵
        self.lifetime = None         # SA有効期限
        self.sequence_number = 0     # シーケンス番号

# SAD（Security Association Database）
sad_example = {
    'outbound_sa': {
        'spi': 0x12345678,
        'dst_ip': '192.168.2.1',
        'protocol': 'ESP',
        'enc_alg': 'AES-256-CBC',
        'auth_alg': 'HMAC-SHA256',
        'lifetime': 3600  # 1時間
    },
    'inbound_sa': {
        'spi': 0x87654321,
        'src_ip': '192.168.2.1',
        'protocol': 'ESP',
        'enc_alg': 'AES-256-CBC',
        'auth_alg': 'HMAC-SHA256',
        'lifetime': 3600
    }
}
```

### Security Policy Database（SPD）
```python
# セキュリティポリシーの例
class SecurityPolicy:
    def __init__(self):
        self.src_ip = None      # 送信元IPアドレス/範囲
        self.dst_ip = None      # 宛先IPアドレス/範囲
        self.protocol = None    # プロトコル（TCP/UDP/ICMP等）
        self.src_port = None    # 送信元ポート
        self.dst_port = None    # 宛先ポート
        self.action = None      # PROTECT/BYPASS/DISCARD

spd_example = [
    {
        'rule_id': 1,
        'src': '192.168.1.0/24',
        'dst': '192.168.2.0/24',
        'protocol': 'any',
        'action': 'PROTECT',
        'ipsec_protocol': 'ESP',
        'mode': 'tunnel'
    },
    {
        'rule_id': 2,
        'src': '192.168.1.0/24',
        'dst': '0.0.0.0/0',
        'protocol': 'HTTP',
        'action': 'BYPASS'
    }
]
```

---

## IPsecのアーキテクチャ

### IPsecプロトコルスタック
```
アプリケーション層    HTTP, SMTP, FTP, etc.
    ↓
トランスポート層      TCP, UDP
    ↓
IPsec層             AH, ESP (セキュリティ処理)
    ↓
ネットワーク層        IP (IPv4/IPv6)
    ↓
データリンク層        Ethernet, WiFi, etc.
    ↓
物理層              物理的な伝送媒体
```

### IPsecの処理フロー
```
送信側:
原IPパケット → SPD参照 → SA選択 → AH/ESP処理 → 暗号化IPパケット → 送信

受信側:
暗号化IPパケット → SA確認 → AH/ESP処理 → 復号化・検証 → 原IPパケット → 上位層
```

### IPsecゲートウェイ構成
```
本社ネットワーク          インターネット          支社ネットワーク
192.168.1.0/24                                   192.168.2.0/24
       |                                               |
   IPsecGW1 ←----------IPsecトンネル----------→ IPsecGW2
  (10.0.1.1)                                    (10.0.2.1)
       |                                               |
   ルーター ←----------暗号化通信----------→     ルーター
       |                                               |
    社内LAN                                        社内LAN
```

---

## IPsecプロトコル

### 1. AH（Authentication Header）

#### AHの機能
- **データ完全性**: パケットの改ざん検出
- **認証**: 送信者の身元確認
- **リプレイ攻撃防止**: シーケンス番号による保護

#### AHヘッダー構造
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Next Header   |  Payload Len  |          RESERVED             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                Security Parameters Index (SPI)                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Sequence Number Field                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                Integrity Check Value-ICV (variable)           |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

#### AHの処理例
```python
import hmac
import hashlib

class AHProcessor:
    def __init__(self, auth_key, auth_algorithm='HMAC-SHA256'):
        self.auth_key = auth_key
        self.auth_algorithm = auth_algorithm
        self.sequence_number = 0
    
    def create_ah_header(self, original_packet, spi):
        self.sequence_number += 1
        
        # AHヘッダー作成
        ah_header = {
            'next_header': 4,  # IPv4
            'payload_length': 4,  # AHヘッダー長
            'reserved': 0,
            'spi': spi,
            'sequence_number': self.sequence_number
        }
        
        # 認証データ計算対象
        auth_data = self._prepare_auth_data(ah_header, original_packet)
        
        # ICV（Integrity Check Value）計算
        icv = hmac.new(
            self.auth_key,
            auth_data,
            hashlib.sha256
        ).digest()[:12]  # 96ビット
        
        ah_header['icv'] = icv
        return ah_header
    
    def verify_ah_packet(self, ah_packet):
        # 受信したAHパケットの検証
        received_icv = ah_packet['ah_header']['icv']
        
        # ICV再計算
        calculated_icv = self._calculate_icv(ah_packet)
        
        return hmac.compare_digest(received_icv, calculated_icv)
```

### 2. ESP（Encapsulating Security Payload）

#### ESPの機能
- **データ機密性**: パケットの暗号化
- **データ完全性**: 改ざん検出
- **認証**: 送信者確認
- **リプレイ攻撃防止**: シーケンス番号管理

#### ESPパケット構造
```
元のIPパケット:
[IP Header][TCP/UDP Header][Data]

ESP処理後:
[IP Header][ESP Header][暗号化されたペイロード][ESP Trailer][ESP Auth]

詳細構造:
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|               Security Parameters Index (SPI)                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Sequence Number                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Payload Data* (variable)                  |
~                                                               ~
|                                                               |
+               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|               |     Padding (0-255 bytes)                    |
+-+-+-+-+-+-+-+-+               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                               |  Pad Length   | Next Header   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Integrity Check Value-ICV   (variable)               |
~                                                               ~
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

* 暗号化対象範囲
```

#### ESP実装例
```python
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.primitives import hashes, hmac
import os

class ESPProcessor:
    def __init__(self, encryption_key, auth_key):
        self.encryption_key = encryption_key
        self.auth_key = auth_key
        self.sequence_number = 0
    
    def encrypt_packet(self, original_packet, spi):
        self.sequence_number += 1
        
        # ESPヘッダー作成
        esp_header = {
            'spi': spi,
            'sequence_number': self.sequence_number
        }
        
        # ペイロードの準備とパディング
        payload = self._prepare_payload(original_packet)
        padded_payload = self._add_padding(payload)
        
        # 暗号化（AES-256-CBC）
        iv = os.urandom(16)  # 初期化ベクトル
        cipher = Cipher(
            algorithms.AES(self.encryption_key),
            modes.CBC(iv)
        )
        encryptor = cipher.encryptor()
        encrypted_payload = encryptor.update(padded_payload) + encryptor.finalize()
        
        # ESP完全パケット作成
        esp_packet = self._build_esp_packet(esp_header, iv, encrypted_payload)
        
        # 認証データ計算
        auth_data = self._calculate_auth_data(esp_packet)
        
        return esp_packet + auth_data
    
    def decrypt_packet(self, esp_packet):
        # ESP構造解析
        esp_header = self._parse_esp_header(esp_packet)
        
        # 認証検証
        if not self._verify_authentication(esp_packet):
            raise Exception("Authentication failed")
        
        # 復号化
        iv = esp_packet[8:24]  # IV抽出
        encrypted_data = esp_packet[24:-12]  # 暗号化データ
        
        cipher = Cipher(
            algorithms.AES(self.encryption_key),
            modes.CBC(iv)
        )
        decryptor = cipher.decryptor()
        decrypted_data = decryptor.update(encrypted_data) + decryptor.finalize()
        
        # パディング除去
        original_packet = self._remove_padding(decrypted_data)
        
        return original_packet
```

---

## 鍵管理システム

### 1. IKE（Internet Key Exchange）

#### IKEv1とIKEv2の比較
```python
ike_comparison = {
    'IKEv1': {
        'phases': 2,
        'phase1': 'ISAKMP SA確立',
        'phase2': 'IPsec SA確立',
        'exchanges': 'Main Mode(6メッセージ) or Aggressive Mode(3メッセージ)',
        'nat_traversal': '拡張として実装',
        'mobike': '非対応',
        'status': '非推奨'
    },
    'IKEv2': {
        'phases': 1,
        'initial_exchange': 'IKE_SA_INIT + IKE_AUTH',
        'exchanges': '4メッセージで完了',
        'nat_traversal': '標準サポート',
        'mobike': '対応（モビリティサポート）',
        'status': '推奨'
    }
}
```

#### IKEv2ネゴシエーション
```
Initiator                    Responder
    |                            |
    | IKE_SA_INIT request        |
    |--------------------------->|
    |                            |
    |        IKE_SA_INIT response|
    |<---------------------------|
    |                            |
    | IKE_AUTH request           |
    |--------------------------->|
    |                            |
    |            IKE_AUTH response|
    |<---------------------------|
    |                            |
    | CREATE_CHILD_SA request    |
    |--------------------------->|
    |                            |
    |   CREATE_CHILD_SA response |
    |<---------------------------|
```

### 2. 事前共有鍵（PSK）vs 証明書認証

#### 事前共有鍵認証
```python
# PSK設定例
psk_config = {
    'authentication_method': 'pre_shared_key',
    'local_id': 'tokyo-office@company.com',
    'remote_id': 'osaka-office@company.com',
    'psk': 'very_long_random_secret_key_123456789',
    'advantages': [
        '設定が簡単',
        '証明書管理が不要',
        '小規模環境に適している'
    ],
    'disadvantages': [
        'スケーラビリティが低い',
        '鍵の配布が困難',
        '鍵の更新が煩雑'
    ]
}
```

#### 証明書認証
```python
# 証明書認証設定例
cert_config = {
    'authentication_method': 'rsa_signature',
    'local_cert': '/etc/ipsec/certs/local.crt',
    'local_key': '/etc/ipsec/private/local.key',
    'ca_cert': '/etc/ipsec/certs/ca.crt',
    'advantages': [
        'スケーラビリティが高い',
        '鍵管理が自動化可能',
        'より強固なセキュリティ',
        '失効管理が可能'
    ],
    'disadvantages': [
        '設定が複雑',
        'PKI基盤が必要',
        '証明書管理のオーバーヘッド'
    ]
}
```

---

## IPsecの動作モード

### 1. トンネルモード

#### 概要と用途
```
用途: サイト間VPN、リモートアクセスVPN
特徴: 元のIPパケット全体を暗号化・カプセル化

パケット構造:
[新IPヘッダー][IPsecヘッダー][暗号化された元IPパケット全体]

利点:
- 元のIPアドレスが隠蔽される
- ゲートウェイ間の接続に最適
- NATトラバーサルが容易
```

#### トンネルモード実装例
```python
class TunnelModeProcessor:
    def __init__(self, tunnel_src_ip, tunnel_dst_ip):
        self.tunnel_src_ip = tunnel_src_ip
        self.tunnel_dst_ip = tunnel_dst_ip
    
    def encapsulate_packet(self, original_packet):
        # 元のIPパケット全体を暗号化
        encrypted_payload = self.encrypt_full_packet(original_packet)
        
        # 新しいIPヘッダー作成
        tunnel_ip_header = {
            'version': 4,
            'header_length': 5,
            'type_of_service': 0,
            'total_length': len(encrypted_payload) + 20,
            'identification': self.generate_id(),
            'flags': 0,
            'fragment_offset': 0,
            'ttl': 64,
            'protocol': 50,  # ESP
            'checksum': 0,
            'src_ip': self.tunnel_src_ip,
            'dst_ip': self.tunnel_dst_ip
        }
        
        return self.build_packet(tunnel_ip_header, encrypted_payload)
    
    def decapsulate_packet(self, tunnel_packet):
        # トンネルIPヘッダー除去
        esp_packet = self.extract_esp_payload(tunnel_packet)
        
        # ESP復号化
        original_packet = self.decrypt_esp_packet(esp_packet)
        
        return original_packet
```

### 2. トランスポートモード

#### 概要と用途
```
用途: ホスト間直接通信
特徴: IPペイロードのみを暗号化

パケット構造:
[元IPヘッダー][IPsecヘッダー][暗号化されたペイロード]

利点:
- オーバーヘッドが小さい
- ホスト間通信に効率的
- IPアドレスが変更されない

制限:
- NAT環境で問題が発生しやすい
- ゲートウェイ間接続には不適切
```

#### トランスポートモード実装例
```python
class TransportModeProcessor:
    def __init__(self):
        pass
    
    def process_packet(self, ip_packet):
        # IPヘッダー解析
        ip_header = self.parse_ip_header(ip_packet)
        payload = self.extract_payload(ip_packet)
        
        # ペイロードのみ暗号化
        encrypted_payload = self.encrypt_payload(payload)
        
        # IPヘッダー更新（プロトコルフィールドをESPに変更）
        ip_header['protocol'] = 50  # ESP
        ip_header['total_length'] = len(ip_header) + len(encrypted_payload)
        
        return self.build_packet(ip_header, encrypted_payload)
```

### 3. モード比較

```python
def mode_comparison():
    return {
        'tunnel_mode': {
            'encapsulation': '元IPパケット全体',
            'overhead': '大きい（新IPヘッダー + IPsecヘッダー）',
            'use_case': 'サイト間VPN、リモートアクセス',
            'nat_compatibility': '良好',
            'security': 'より高い（元IPアドレス隠蔽）'
        },
        'transport_mode': {
            'encapsulation': 'ペイロードのみ',
            'overhead': '小さい（IPsecヘッダーのみ）',
            'use_case': 'ホスト間直接通信',
            'nat_compatibility': '問題あり',
            'security': '標準的'
        }
    }
```

---

## VPN実装

### 1. サイト間VPN設定

#### strongSwan設定例
```bash
# /etc/ipsec.conf
config setup
    charondebug="ike 1, knl 1, cfg 0"
    uniqueids=no

conn site-to-site
    auto=start
    compress=no
    type=tunnel
    keyexchange=ikev2
    fragmentation=yes
    forceencaps=yes
    
    # 認証設定
    authby=secret
    
    # ローカル設定（本社）
    left=203.0.113.1
    leftid=@tokyo-office
    leftsubnet=192.168.1.0/24
    leftauth=psk
    
    # リモート設定（支社）
    right=203.0.113.2
    rightid=@osaka-office
    rightsubnet=192.168.2.0/24
    rightauth=psk
    
    # 暗号化設定
    ike=chacha20poly1305-sha256-curve25519-prfsha256,aes256-sha1-modp1024!
    esp=chacha20poly1305-sha256,aes256-sha256-modp1024!
    
    # SA有効期限
    ikelifetime=10800s
    keylife=3600s
    rekeymargin=540s
    keyingtries=1
    
    # DPD（Dead Peer Detection）
    dpdaction=restart
    dpddelay=30s
    dpdtimeout=120s

# /etc/ipsec.secrets
@tokyo-office @osaka-office : PSK "very_long_random_secret_key_123456789"
```

#### 設定確認コマンド
```bash
# IPsec設定確認
sudo ipsec statusall

# SA状態確認
sudo ipsec status

# 接続テスト
sudo ipsec up site-to-site

# ログ確認
sudo journalctl -u strongswan -f
```

### 2. リモートアクセスVPN

#### サーバー側設定
```bash
# /etc/ipsec.conf
conn roadwarrior
    auto=add
    compress=no
    type=tunnel
    keyexchange=ikev2
    fragmentation=yes
    forceencaps=yes
    
    # サーバー設定
    left=%any
    leftid=@vpn.company.com
    leftcert=server.crt
    leftsendcert=always
    leftsubnet=0.0.0.0/0
    
    # クライアント設定
    right=%any
    rightauth=eap-mschapv2
    rightsourceip=10.10.10.0/24
    rightdns=8.8.8.8,8.8.4.4
    
    # 暗号化設定
    ike=aes256-sha256-modp1024!
    esp=aes256-sha256!
    
    # EAP設定
    eap_identity=%identity

# /etc/ipsec.secrets
: RSA server.key
user1 : EAP "password123"
user2 : EAP "password456"
```

#### クライアント側設定
```bash
# /etc/ipsec.conf
conn company-vpn
    auto=start
    compress=no
    type=tunnel
    keyexchange=ikev2
    
    # サーバー設定
    right=vpn.company.com
    rightid=@vpn.company.com
    rightca="C=JP, O=Company, CN=Company CA"
    
    # クライアント設定
    left=%defaultroute
    leftauth=eap
    leftsourceip=%config
    leftfirewall=yes
    
    # 暗号化設定
    ike=aes256-sha256-modp1024!
    esp=aes256-sha256!
    
    # EAP設定
    eap_identity=user1

# /etc/ipsec.secrets
user1 : EAP "password123"
```

### 3. 高可用性VPN構成

#### アクティブ・スタンバイ構成
```python
class HighAvailabilityVPN:
    def __init__(self):
        self.primary_gateway = "203.0.113.1"
        self.secondary_gateway = "203.0.113.2"
        self.virtual_ip = "203.0.113.100"
        self.health_check_interval = 30
    
    def configure_ha_vpn(self):
        return {
            'primary_config': {
                'gateway_ip': self.primary_gateway,
                'priority': 100,
                'preempt': True,
                'track_interface': 'eth0',
                'track_routes': ['0.0.0.0/0']
            },
            'secondary_config': {
                'gateway_ip': self.secondary_gateway,
                'priority': 90,
                'preempt': False,
                'track_interface': 'eth0',
                'track_routes': ['0.0.0.0/0']
            },
            'vrrp_config': {
                'virtual_router_id': 1,
                'virtual_ip': self.virtual_ip,
                'advertisement_interval': 1,
                'authentication': 'password123'
            }
        }
    
    def monitor_vpn_health(self):
        """VPNヘルスチェック"""
        health_checks = [
            self.check_ipsec_tunnels(),
            self.check_routing_table(),
            self.check_gateway_connectivity(),
            self.check_peer_reachability()
        ]
        
        return all(health_checks)
```

---

## セキュリティ考慮事項

### 1. 暗号化アルゴリズムの選択

#### 推奨アルゴリズム（2024年基準）
```python
recommended_algorithms = {
    'ike_encryption': [
        'AES-256-GCM',      # 最推奨
        'AES-256-CBC',      # 標準
        'ChaCha20-Poly1305' # モダンな選択肢
    ],
    'esp_encryption': [
        'AES-256-GCM',      # AEAD暗号（推奨）
        'AES-256-CBC',      # 従来型
        'ChaCha20-Poly1305' # 高性能
    ],
    'integrity': [
        'HMAC-SHA256',      # 標準
        'HMAC-SHA384',      # より強固
        'HMAC-SHA512'       # 最強
    ],
    'dh_groups': [
        'Curve25519',       # 楕円曲線（推奨）
        'DH Group 19',      # 256-bit ECP
        'DH Group 20',      # 384-bit ECP
        'DH Group 21'       # 521-bit ECP
    ]
}

deprecated_algorithms = {
    'avoid': [
        'DES', '3DES',      # 弱い暗号化
        'MD5', 'SHA1',      # 弱いハッシュ
        'DH Group 1-2',     # 弱いDHグループ
        'RC4'               # 脆弱な暗号
    ]
}
```

### 2. 鍵管理のベストプラクティス

#### 鍵のライフサイクル管理
```python
class KeyLifecycleManager:
    def __init__(self):
        self.key_lifetime_recommendations = {
            'ike_sa_lifetime': {
                'recommended': 86400,    # 24時間
                'maximum': 604800,       # 7日
                'minimum': 3600          # 1時間
            },
            'ipsec_sa_lifetime': {
                'recommended': 3600,     # 1時間
                'maximum': 86400,        # 24時間
                'minimum': 300           # 5分
            },
            'rekey_margin': {
                'recommended': 540,      # 9分
                'description': 'SA期限切れ前に再鍵交換開始'
            }
        }
    
    def calculate_optimal_lifetime(self, traffic_volume, security_level):
        """トラフィック量とセキュリティレベルに基づく最適なSA有効期限計算"""
        base_lifetime = 3600  # 1時間
        
        # トラフィック量による調整
        if traffic_volume > 1000000000:  # 1GB以上
            traffic_factor = 0.5
        elif traffic_volume > 100000000:  # 100MB以上
            traffic_factor = 0.75
        else:
            traffic_factor = 1.0
        
        # セキュリティレベルによる調整
        security_factor = {
            'low': 2.0,
            'medium': 1.0,
            'high': 0.5,
            'critical': 0.25
        }.get(security_level, 1.0)
        
        optimal_lifetime = int(base_lifetime * traffic_factor * security_factor)
        
        # 最小・最大値でクランプ
        return max(300, min(86400, optimal_lifetime))
```

### 3. Perfect Forward Secrecy（PFS）

#### PFS実装の重要性
```python
def pfs_explanation():
    return {
        'definition': 'セッション鍵が漏洩しても過去の通信が解読されない性質',
        'implementation': 'Diffie-Hellman鍵交換を使用',
        'benefits': [
            '長期鍵の漏洩による被害を限定',
            '過去の通信記録の安全性確保',
            '法執行機関等による大量監視への対抗'
        ],
        'configuration': {
            'strongswan': 'pfs=yes',
            'cisco': 'crypto ipsec transform-set ... ah-sha-hmac esp-aes esp-sha-hmac',
            'pfsgroup': 'group14以上を推奨'
        }
    }
```

---

## 実際のケーススタディ

### ケース1: 大企業の拠点間VPN構築

**要件**:
- 本社と50の支社を接続
- 高可用性が必要
- 帯域制御が必要

**実装**:
```yaml
# Hub-and-Spoke構成
vpn_topology:
  type: "hub-and-spoke"
  hub:
    location: "Tokyo HQ"
    primary_gateway: "203.0.113.1"
    secondary_gateway: "203.0.113.2"
    bandwidth: "1Gbps"
  
  spokes:
    - name: "Osaka Branch"
      gateway: "203.0.113.10"
      subnet: "192.168.10.0/24"
      bandwidth: "100Mbps"
    - name: "Nagoya Branch"
      gateway: "203.0.113.11"
      subnet: "192.168.11.0/24"
      bandwidth: "100Mbps"
    # ... 他48拠点

security_policy:
  encryption: "AES-256-GCM"
  integrity: "HMAC-SHA256"
  dh_group: "Curve25519"
  pfs: true
  sa_lifetime: 3600
  dpd_enabled: true
```

**結果**:
- 全拠点接続成功率: 99.9%
- 平均接続確立時間: 3秒
- 帯域使用効率: 95%

### ケース2: クラウド接続でのIPsec実装

**シナリオ**: AWS VPCとオンプレミスの接続

**設定例**:
```python
# AWS VPN Gateway設定
aws_vpn_config = {
    'customer_gateway': {
        'bgp_asn': 65000,
        'ip_address': '203.0.113.1',
        'type': 'ipsec.1'
    },
    'vpn_connection': {
        'type': 'ipsec.1',
        'customer_gateway_id': 'cgw-12345678',
        'vpn_gateway_id': 'vgw-87654321',
        'static_routes_only': False
    },
    'tunnel_options': {
        'tunnel_inside_cidr': '169.254.10.0/30',
        'pre_shared_key': 'generated_by_aws',
        'ike_versions': ['ikev1', 'ikev2'],
        'phase1_encryption_algorithms': ['AES256'],
        'phase1_integrity_algorithms': ['SHA256'],
        'phase1_dh_group_numbers': [14],
        'phase2_encryption_algorithms': ['AES256'],
        'phase2_integrity_algorithms': ['SHA256'],
        'phase2_dh_group_numbers': [14]
    }
}

# オンプレミス側strongSwan設定
onprem_config = """
conn aws-vpn-tunnel1
    auto=start
    left=%defaultroute
    leftid=203.0.113.1
    leftsubnet=192.168.1.0/24
    right=52.1.2.3
    rightsubnet=10.0.0.0/16
    authby=secret
    keyexchange=ikev1
    ike=aes256-sha256-modp2048!
    esp=aes256-sha256-modp2048!
    keyingtries=%forever
    ikelifetime=28800s
    salifetime=3600s
    dpddelay=10s
    dpdtimeout=30s
    dpdaction=restart
"""
```

### ケース3: リモートワーク環境でのIPsec VPN

**課題**: COVID-19によるリモートワーク急増への対応

**解決策**:
```python
class ScalableRemoteVPN:
    def __init__(self):
        self.max_concurrent_users = 1000
        self.load_balancers = [
            '203.0.113.10',
            '203.0.113.11',
            '203.0.113.12'
        ]
    
    def configure_load_balancing(self):
        return {
            'method': 'round_robin_with_health_check',
            'health_check': {
                'interval': 30,
                'timeout': 10,
                'retries': 3
            },
            'session_persistence': 'source_ip',
            'ssl_offloading': True
        }
    
    def auto_scaling_policy(self):
        return {
            'scale_up_threshold': 80,    # CPU使用率80%
            'scale_down_threshold': 30,  # CPU使用率30%
            'cooldown_period': 300,      # 5分
            'max_instances': 10,
            'min_instances': 2
        }
```

**成果**:
- 同時接続ユーザー数: 1,500人
- 接続成功率: 98.5%
- 平均スループット: 50Mbps/ユーザー
- セキュリティインシデント: 0件

---

## まとめ

### IPsecの重要ポイント

1. **包括的なセキュリティ**: 機密性、完全性、認証を同時に提供
2. **透過的な実装**: アプリケーション層への影響なし
3. **柔軟な構成**: トンネル/トランスポートモード、様々な暗号化オプション
4. **標準化**: RFC準拠による相互運用性

### 実装時の推奨事項

- **強力な暗号化**: AES-256、SHA-256以上を使用
- **Perfect Forward Secrecy**: 必ず有効化
- **適切なSA有効期限**: トラフィック量に応じた調整
- **継続的な監視**: 接続状態とセキュリティログの監視

### 今後の展望

- **Post-Quantum Cryptography**: 量子コンピュータ耐性暗号への移行
- **WireGuard統合**: より軽量で高速なVPNプロトコルとの併用
- **SD-WAN連携**: ソフトウェア定義ネットワークとの統合
- **ゼロトラストアーキテクチャ**: より細かな認証・認可の実装

IPsecは、ネットワークセキュリティの基盤技術として、今後も重要な役割を果たし続けます。適切な理解と実装により、安全で信頼性の高いネットワーク通信を実現できます。

---

**参考資料**:
- RFC 4301: Security Architecture for the Internet Protocol
- RFC 4302: IP Authentication Header
- RFC 4303: IP Encapsulating Security Payload (ESP)
- RFC 7296: Internet Key Exchange Protocol Version 2 (IKEv2)
- NIST SP 800-77 Rev. 1: Guide to IPsec VPNs

