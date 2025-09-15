# AES入門

## AESとは

**AES（Advanced Encryption Standard）**は、アメリカ国立標準技術研究所（NIST）が2001年に制定した対称暗号化アルゴリズムです。現在、世界で最も広く使用されている暗号化技術の一つで、政府機関から民間企業まで、あらゆる場面でデータの機密性を保護するために使用されています。

## なぜAESが必要なのか

### 問題
- **データの機密性**: 重要な情報を第三者に知られずに保護する必要
- **通信の安全性**: ネットワーク経由でのデータ送信の保護
- **ストレージの安全性**: 保存されたデータの保護
- **標準化の必要性**: 統一された暗号化方式の需要

### 解決策
AESを使用することで、強力で効率的な暗号化により、データの機密性を確保できます。

## AESの基本仕様

### 1. 主要な特徴
| 項目 | 仕様 |
|------|------|
| **暗号化方式** | 対称暗号（共通鍵暗号） |
| **ブロックサイズ** | 128ビット（16バイト） |
| **鍵長** | 128、192、256ビット |
| **ラウンド数** | 10、12、14ラウンド |
| **モード** | ECB、CBC、CFB、OFB、CTR、GCM |

### 2. 鍵長とラウンド数の関係
| 鍵長 | ラウンド数 | セキュリティレベル |
|------|-----------|-------------------|
| **128ビット** | 10ラウンド | 高 |
| **192ビット** | 12ラウンド | 非常に高 |
| **256ビット** | 14ラウンド | 最高 |

## AESの暗号化プロセス

### 1. 基本的な流れ
```
[平文] → [鍵] → [AES暗号化] → [暗号文]
[暗号文] → [鍵] → [AES復号化] → [平文]
```

### 2. ラウンド処理
```
[入力] → [SubBytes] → [ShiftRows] → [MixColumns] → [AddRoundKey] → [出力]
```

### 3. 各処理の説明
| 処理 | 説明 | 役割 |
|------|------|------|
| **SubBytes** | バイト置換 | 非線形変換 |
| **ShiftRows** | 行シフト | 拡散効果 |
| **MixColumns** | 列混合 | 拡散効果 |
| **AddRoundKey** | 鍵加算 | 鍵の適用 |

## AESのモード

### 1. ECB（Electronic Codebook）
```
[ブロック1] → [AES] → [暗号文1]
[ブロック2] → [AES] → [暗号文2]
[ブロック3] → [AES] → [暗号文3]
```

**特徴**:
- 各ブロックを独立して暗号化
- 並列処理が可能
- 同じ平文は同じ暗号文になる（脆弱性）

### 2. CBC（Cipher Block Chaining）
```
[ブロック1] → [AES] → [暗号文1] → [次のブロックのIV]
[ブロック2] → [XOR] → [AES] → [暗号文2]
[ブロック3] → [XOR] → [AES] → [暗号文3]
```

**特徴**:
- 前のブロックの暗号文を次のブロックに影響
- 初期化ベクトル（IV）が必要
- 並列処理が困難

### 3. CTR（Counter）
```
[カウンタ1] → [AES] → [キーストリーム1] → [XOR] → [暗号文1]
[カウンタ2] → [AES] → [キーストリーム2] → [XOR] → [暗号文2]
[カウンタ3] → [AES] → [キーストリーム3] → [XOR] → [暗号文3]
```

**特徴**:
- ストリーム暗号として動作
- 並列処理が可能
- 認証機能なし

### 4. GCM（Galois/Counter Mode）
```
[カウンタ] → [AES] → [キーストリーム] → [XOR] → [暗号文]
[認証データ] → [GHASH] → [認証タグ]
```

**特徴**:
- 暗号化と認証を同時に実行
- 認証タグで改ざん検知
- 高速処理が可能

## 実装例

### 1. Pythonでの実装
```python
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.backends import default_backend
import os

def aes_encrypt(plaintext, key):
    """AES暗号化"""
    # 初期化ベクトル（IV）の生成
    iv = os.urandom(16)  # 128ビット
    
    # AES暗号化器の作成
    cipher = Cipher(
        algorithms.AES(key),
        modes.CBC(iv),
        backend=default_backend()
    )
    
    # 暗号化の実行
    encryptor = cipher.encryptor()
    ciphertext = encryptor.update(plaintext) + encryptor.finalize()
    
    return iv + ciphertext

def aes_decrypt(ciphertext, key):
    """AES復号化"""
    # IVと暗号文を分離
    iv = ciphertext[:16]
    ciphertext = ciphertext[16:]
    
    # AES復号化器の作成
    cipher = Cipher(
        algorithms.AES(key),
        modes.CBC(iv),
        backend=default_backend()
    )
    
    # 復号化の実行
    decryptor = cipher.decryptor()
    plaintext = decryptor.update(ciphertext) + decryptor.finalize()
    
    return plaintext

# 使用例
key = os.urandom(32)  # 256ビット鍵
plaintext = b"Hello, World! This is a test message."

# 暗号化
ciphertext = aes_encrypt(plaintext, key)
print(f"暗号文: {ciphertext.hex()}")

# 復号化
decrypted = aes_decrypt(ciphertext, key)
print(f"復号文: {decrypted.decode()}")
```

### 2. GCMモードでの実装
```python
def aes_gcm_encrypt(plaintext, key, associated_data=b""):
    """AES-GCM暗号化（認証付き）"""
    # 初期化ベクトル（IV）の生成
    iv = os.urandom(12)  # 96ビット（GCM推奨）
    
    # AES-GCM暗号化器の作成
    cipher = Cipher(
        algorithms.AES(key),
        modes.GCM(iv),
        backend=default_backend()
    )
    
    # 暗号化の実行
    encryptor = cipher.encryptor()
    encryptor.authenticate_additional_data(associated_data)
    ciphertext = encryptor.update(plaintext) + encryptor.finalize()
    
    return iv + ciphertext + encryptor.tag

def aes_gcm_decrypt(ciphertext, key, associated_data=b""):
    """AES-GCM復号化（認証付き）"""
    # IV、暗号文、タグを分離
    iv = ciphertext[:12]
    tag = ciphertext[-16:]
    ciphertext = ciphertext[12:-16]
    
    # AES-GCM復号化器の作成
    cipher = Cipher(
        algorithms.AES(key),
        modes.GCM(iv, tag),
        backend=default_backend()
    )
    
    # 復号化の実行
    decryptor = cipher.decryptor()
    decryptor.authenticate_additional_data(associated_data)
    plaintext = decryptor.update(ciphertext) + decryptor.finalize()
    
    return plaintext

# 使用例
key = os.urandom(32)  # 256ビット鍵
plaintext = b"Secret message"
associated_data = b"metadata"

# 暗号化
ciphertext = aes_gcm_encrypt(plaintext, key, associated_data)
print(f"暗号文: {ciphertext.hex()}")

# 復号化
decrypted = aes_gcm_decrypt(ciphertext, key, associated_data)
print(f"復号文: {decrypted.decode()}")
```

### 3. OpenSSLでの実装
```bash
# AES-256-CBCでの暗号化
openssl enc -aes-256-cbc -in plaintext.txt -out ciphertext.enc -k password

# AES-256-CBCでの復号化
openssl enc -aes-256-cbc -d -in ciphertext.enc -out plaintext.txt -k password

# AES-256-GCMでの暗号化
openssl enc -aes-256-gcm -in plaintext.txt -out ciphertext.enc -k password

# 鍵ファイルを使用した暗号化
openssl enc -aes-256-cbc -in plaintext.txt -out ciphertext.enc -pass file:keyfile
```

## セキュリティ考慮事項

### 1. 鍵管理
```python
# 安全な鍵生成
import secrets

def generate_secure_key(key_length=32):
    """安全な鍵を生成"""
    return secrets.token_bytes(key_length)

# 鍵の保存（暗号化）
def encrypt_key(key, master_password):
    """鍵を暗号化して保存"""
    # マスターキーで鍵を暗号化
    encrypted_key = aes_encrypt(key, master_password)
    return encrypted_key

# 鍵の復元
def decrypt_key(encrypted_key, master_password):
    """暗号化された鍵を復元"""
    return aes_decrypt(encrypted_key, master_password)
```

### 2. 初期化ベクトル（IV）
```python
# 安全なIV生成
def generate_secure_iv():
    """暗号学的に安全なIVを生成"""
    return os.urandom(16)  # 128ビット

# IVの再利用を防ぐ
def encrypt_with_unique_iv(plaintext, key):
    """一意のIVで暗号化"""
    iv = generate_secure_iv()
    # 暗号化処理...
    return iv + ciphertext
```

### 3. パディング
```python
# PKCS7パディングの実装
def pkcs7_pad(data, block_size):
    """PKCS7パディングを適用"""
    padding_length = block_size - (len(data) % block_size)
    padding = bytes([padding_length] * padding_length)
    return data + padding

def pkcs7_unpad(data):
    """PKCS7パディングを除去"""
    padding_length = data[-1]
    return data[:-padding_length]
```

## パフォーマンス最適化

### 1. ハードウェア支援
```python
# AES-NIの確認
import platform

def check_aes_ni_support():
    """AES-NIサポートを確認"""
    try:
        import cpuinfo
        cpu_info = cpuinfo.get_cpu_info()
        return 'aes' in cpu_info.get('flags', [])
    except:
        return False

# パフォーマンステスト
import time

def performance_test():
    """AES暗号化のパフォーマンスをテスト"""
    key = os.urandom(32)
    data = os.urandom(1024 * 1024)  # 1MB
    
    start_time = time.time()
    for _ in range(100):
        aes_encrypt(data, key)
    end_time = time.time()
    
    print(f"暗号化速度: {100 / (end_time - start_time):.2f} MB/s")
```

### 2. 並列処理
```python
from concurrent.futures import ThreadPoolExecutor
import threading

def parallel_encrypt(data_blocks, key):
    """並列でAES暗号化を実行"""
    def encrypt_block(block):
        return aes_encrypt(block, key)
    
    with ThreadPoolExecutor(max_workers=4) as executor:
        results = list(executor.map(encrypt_block, data_blocks))
    
    return results
```

## 実際の使用例

### 1. ファイル暗号化
```python
def encrypt_file(input_file, output_file, key):
    """ファイルを暗号化"""
    with open(input_file, 'rb') as f:
        plaintext = f.read()
    
    ciphertext = aes_encrypt(plaintext, key)
    
    with open(output_file, 'wb') as f:
        f.write(ciphertext)

def decrypt_file(input_file, output_file, key):
    """ファイルを復号化"""
    with open(input_file, 'rb') as f:
        ciphertext = f.read()
    
    plaintext = aes_decrypt(ciphertext, key)
    
    with open(output_file, 'wb') as f:
        f.write(plaintext)
```

### 2. データベース暗号化
```python
import sqlite3

def encrypt_database_field(value, key):
    """データベースフィールドを暗号化"""
    if value is None:
        return None
    return aes_encrypt(value.encode(), key)

def decrypt_database_field(encrypted_value, key):
    """データベースフィールドを復号化"""
    if encrypted_value is None:
        return None
    return aes_decrypt(encrypted_value, key).decode()
```

## まとめ

AESは、現代の暗号化において最も重要な技術の一つです。適切な実装と運用により、強力なデータ保護を実現できます。

### 次のステップ
- 鍵管理システムの構築
- 認証付き暗号化の実装
- パフォーマンス最適化
- セキュリティ監査の実施
