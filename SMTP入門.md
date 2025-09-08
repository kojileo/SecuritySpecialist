# SMTP入門

## 目次
1. [SMTPとは](#smtpとは)
2. [SMTPの基本概念](#smtpの基本概念)
3. [SMTPの仕組み](#smtpの仕組み)
4. [SMTPコマンドとレスポンス](#smtpコマンドとレスポンス)
5. [SMTP認証](#smtp認証)
6. [SMTP over SSL/TLS](#smtp-over-ssltls)
7. [メールヘッダーとフォーマット](#メールヘッダーとフォーマット)
8. [SMTPサーバーの設定](#smtpサーバーの設定)
9. [セキュリティ上の脅威](#セキュリティ上の脅威)
10. [対策と防御](#対策と防御)
11. [トラブルシューティング](#トラブルシューティング)
12. [実際のケーススタディ](#実際のケーススタディ)
13. [まとめ](#まとめ)

---

## SMTPとは

**SMTP（Simple Mail Transfer Protocol）** は、インターネット上で電子メールを送信するための標準プロトコルです。メールサーバー間でのメール転送や、メールクライアントからメールサーバーへのメール送信に使用されます。

### 基本概念
- **メール送信プロトコル**: 電子メールの配送を担当
- **プッシュ型プロトコル**: 送信者がメールを相手のサーバーに押し込む
- **TCP接続**: 信頼性のある接続を使用（通常ポート25、587、465）
- **テキストベース**: 人間が読める形式のコマンド

### SMTPの役割
```
メール送信の流れ:
送信者 → [メールクライアント] → [SMTPサーバー] → [受信者のSMTPサーバー] → 受信者

具体例:
alice@company.com → Outlook → mail.company.com → mail.example.com → bob@example.com
```

### 他のメールプロトコルとの関係
```python
mail_protocols = {
    'SMTP': {
        'purpose': 'メール送信',
        'direction': 'クライアント → サーバー、サーバー → サーバー',
        'ports': [25, 587, 465],
        'security': 'STARTTLS, SMTPS'
    },
    'POP3': {
        'purpose': 'メール受信（ダウンロード）',
        'direction': 'サーバー → クライアント',
        'ports': [110, 995],
        'security': 'POP3S'
    },
    'IMAP': {
        'purpose': 'メール受信（同期）',
        'direction': 'サーバー ⇄ クライアント',
        'ports': [143, 993],
        'security': 'IMAPS'
    }
}
```

---

## SMTPの基本概念

### メール配送の仕組み

#### MTA（Mail Transfer Agent）
```
MTA = メールを転送するサーバー

主要なMTAソフトウェア:
├── Postfix（最も一般的）
├── Sendmail（歴史的に重要）
├── Exim（Debian系で使用）
├── qmail（セキュリティ重視）
└── Microsoft Exchange（Windows環境）
```

#### メール配送プロセス
```python
class MailDeliveryProcess:
    def __init__(self):
        self.steps = [
            "1. MUA（Mail User Agent）がメール作成",
            "2. MUA が MSA（Mail Submission Agent）にメール送信",
            "3. MSA がメール検証・処理",
            "4. MSA が MTA（Mail Transfer Agent）に転送",
            "5. MTA が DNS でMXレコード検索",
            "6. MTA が宛先MTAに接続",
            "7. 宛先MTA がメール受信・保存",
            "8. MDA（Mail Delivery Agent）がメールボックスに配送"
        ]
    
    def get_components(self):
        return {
            'MUA': 'Outlook, Thunderbird, Gmail web interface',
            'MSA': 'メール送信エージェント（通常MTAと同じサーバー）',
            'MTA': 'Postfix, Sendmail, Exchange',
            'MDA': 'Dovecot, Courier, Exchange'
        }
```

### DNSとメール配送

#### MXレコード
```bash
# MXレコード例
$ dig MX example.com

example.com.    3600    IN    MX    10 mail.example.com.
example.com.    3600    IN    MX    20 mail2.example.com.
example.com.    3600    IN    MX    30 backup-mail.example.com.

# 優先度の解釈
# 数値が小さいほど優先度が高い
# 10 = 最優先メールサーバー
# 20 = セカンダリメールサーバー
# 30 = バックアップメールサーバー
```

#### SPF、DKIM、DMARCレコード
```bash
# SPF（Sender Policy Framework）レコード
example.com.    TXT    "v=spf1 mx include:_spf.google.com ~all"

# DKIM（DomainKeys Identified Mail）レコード
default._domainkey.example.com.    TXT    "v=DKIM1; k=rsa; p=MIGfMA0GCS..."

# DMARC（Domain-based Message Authentication, Reporting & Conformance）レコード
_dmarc.example.com.    TXT    "v=DMARC1; p=quarantine; rua=mailto:dmarc@example.com"
```

---

## SMTPの仕組み

### SMTP接続の確立

#### 基本的なSMTPセッション
```
Client                          Server
  |                               |
  | TCP接続確立（ポート25）        |
  |------------------------------>|
  |                               |
  |           220 Ready           |
  |<------------------------------|
  |                               |
  | HELO client.example.com       |
  |------------------------------>|
  |                               |
  |    250 Hello client.example.com|
  |<------------------------------|
  |                               |
  | MAIL FROM:<alice@example.com> |
  |------------------------------>|
  |                               |
  |           250 OK              |
  |<------------------------------|
  |                               |
  | RCPT TO:<bob@destination.com> |
  |------------------------------>|
  |                               |
  |           250 OK              |
  |<------------------------------|
  |                               |
  | DATA                          |
  |------------------------------>|
  |                               |
  |    354 Start mail input       |
  |<------------------------------|
  |                               |
  | [メール本文]                   |
  | .                             |
  |------------------------------>|
  |                               |
  |     250 Message accepted      |
  |<------------------------------|
  |                               |
  | QUIT                          |
  |------------------------------>|
  |                               |
  |      221 Bye                  |
  |<------------------------------|
```

### Python実装例

#### 基本的なSMTP送信
```python
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from email.mime.base import MIMEBase
from email import encoders
import ssl

class SMTPClient:
    def __init__(self, smtp_server, port, username=None, password=None):
        self.smtp_server = smtp_server
        self.port = port
        self.username = username
        self.password = password
        self.connection = None
    
    def connect(self, use_tls=True):
        """SMTPサーバーに接続"""
        try:
            if self.port == 465:
                # SMTPS（SSL/TLS暗号化）
                context = ssl.create_default_context()
                self.connection = smtplib.SMTP_SSL(self.smtp_server, self.port, context=context)
            else:
                # 通常のSMTP
                self.connection = smtplib.SMTP(self.smtp_server, self.port)
                
                if use_tls:
                    # STARTTLS使用
                    self.connection.starttls()
            
            # 認証が必要な場合
            if self.username and self.password:
                self.connection.login(self.username, self.password)
            
            return True
            
        except Exception as e:
            print(f"SMTP接続エラー: {e}")
            return False
    
    def send_email(self, from_addr, to_addrs, subject, body, attachments=None):
        """メール送信"""
        if not self.connection:
            if not self.connect():
                return False
        
        try:
            # メッセージ作成
            msg = MIMEMultipart()
            msg['From'] = from_addr
            msg['To'] = ', '.join(to_addrs) if isinstance(to_addrs, list) else to_addrs
            msg['Subject'] = subject
            
            # 本文追加
            msg.attach(MIMEText(body, 'plain', 'utf-8'))
            
            # 添付ファイル処理
            if attachments:
                for file_path in attachments:
                    self.add_attachment(msg, file_path)
            
            # メール送信
            text = msg.as_string()
            self.connection.sendmail(from_addr, to_addrs, text)
            
            return True
            
        except Exception as e:
            print(f"メール送信エラー: {e}")
            return False
    
    def add_attachment(self, msg, file_path):
        """添付ファイル追加"""
        try:
            with open(file_path, "rb") as attachment:
                part = MIMEBase('application', 'octet-stream')
                part.set_payload(attachment.read())
            
            encoders.encode_base64(part)
            part.add_header(
                'Content-Disposition',
                f'attachment; filename= {os.path.basename(file_path)}'
            )
            
            msg.attach(part)
            
        except Exception as e:
            print(f"添付ファイルエラー: {e}")
    
    def disconnect(self):
        """接続終了"""
        if self.connection:
            self.connection.quit()
            self.connection = None

# 使用例
smtp_client = SMTPClient('smtp.gmail.com', 587, 'user@gmail.com', 'password')

if smtp_client.connect():
    success = smtp_client.send_email(
        from_addr='sender@example.com',
        to_addrs=['recipient@example.com'],
        subject='テストメール',
        body='これはテストメールです。',
        attachments=['document.pdf']
    )
    
    if success:
        print("メール送信成功")
    else:
        print("メール送信失敗")
    
    smtp_client.disconnect()
```

---

## SMTPコマンドとレスポンス

### 主要なSMTPコマンド

#### 基本コマンド
```python
smtp_commands = {
    'HELO': {
        'purpose': 'クライアントの識別（古い形式）',
        'syntax': 'HELO hostname',
        'example': 'HELO mail.example.com',
        'response': '250'
    },
    'EHLO': {
        'purpose': 'クライアントの識別（拡張形式）',
        'syntax': 'EHLO hostname',
        'example': 'EHLO mail.example.com',
        'response': '250-hostname\n250-STARTTLS\n250-AUTH LOGIN PLAIN\n250 SIZE 52428800'
    },
    'MAIL FROM': {
        'purpose': '送信者アドレス指定',
        'syntax': 'MAIL FROM:<email>',
        'example': 'MAIL FROM:<alice@example.com>',
        'response': '250'
    },
    'RCPT TO': {
        'purpose': '受信者アドレス指定',
        'syntax': 'RCPT TO:<email>',
        'example': 'RCPT TO:<bob@example.com>',
        'response': '250'
    },
    'DATA': {
        'purpose': 'メール本文開始',
        'syntax': 'DATA',
        'example': 'DATA',
        'response': '354'
    },
    'QUIT': {
        'purpose': 'セッション終了',
        'syntax': 'QUIT',
        'example': 'QUIT',
        'response': '221'
    }
}
```

#### 拡張コマンド
```python
extended_commands = {
    'STARTTLS': {
        'purpose': 'TLS暗号化開始',
        'syntax': 'STARTTLS',
        'response': '220 Ready to start TLS'
    },
    'AUTH': {
        'purpose': 'SMTP認証',
        'syntax': 'AUTH mechanism',
        'mechanisms': ['PLAIN', 'LOGIN', 'CRAM-MD5', 'DIGEST-MD5'],
        'example': 'AUTH PLAIN dXNlcm5hbWUAdXNlcm5hbWUAcGFzc3dvcmQ='
    },
    'VRFY': {
        'purpose': 'メールアドレス検証',
        'syntax': 'VRFY <email>',
        'example': 'VRFY alice@example.com',
        'security_note': 'セキュリティ上通常無効化'
    },
    'EXPN': {
        'purpose': 'メーリングリスト展開',
        'syntax': 'EXPN <list>',
        'example': 'EXPN staff',
        'security_note': 'セキュリティ上通常無効化'
    }
}
```

### SMTPレスポンスコード

#### レスポンスコード分類
```python
class SMTPResponseCodes:
    def __init__(self):
        self.codes = {
            # 2xx: 成功
            '220': 'サービス準備完了',
            '221': '接続終了',
            '250': 'コマンド成功',
            '251': 'ユーザーローカルではないが転送する',
            '252': 'ユーザー検証不可だが受け入れる',
            
            # 3xx: 追加情報が必要
            '354': 'メールデータ入力開始',
            
            # 4xx: 一時的エラー
            '421': 'サービス利用不可（一時的）',
            '450': 'メールアクション未実行（メールボックス使用中）',
            '451': 'アクション中断（処理エラー）',
            '452': 'アクション未実行（容量不足）',
            
            # 5xx: 恒久的エラー
            '500': 'コマンド認識不可',
            '501': 'パラメータエラー',
            '502': 'コマンド未実装',
            '503': 'コマンド順序エラー',
            '504': 'パラメータ未実装',
            '550': 'メールボックス利用不可',
            '551': 'ユーザーローカルではない',
            '552': '容量制限超過',
            '553': 'メールアドレス不正',
            '554': 'トランザクション失敗'
        }
    
    def get_category(self, code):
        first_digit = code[0]
        categories = {
            '2': 'Success',
            '3': 'Intermediate',
            '4': 'Temporary Failure',
            '5': 'Permanent Failure'
        }
        return categories.get(first_digit, 'Unknown')
    
    def is_temporary_error(self, code):
        return code.startswith('4')
    
    def is_permanent_error(self, code):
        return code.startswith('5')
```

---

## SMTP認証

### 認証の必要性
```
認証なしSMTP（ポート25）:
- オープンリレー問題
- スパム送信の踏み台
- 現在は通常無効化

認証ありSMTP（ポート587）:
- 正当なユーザーのみ送信可能
- スパム対策
- ISPによるブロック回避
```

### 認証メカニズム

#### PLAIN認証
```python
import base64

def smtp_auth_plain(username, password):
    """PLAIN認証の実装"""
    # フォーマット: \0username\0password
    auth_string = f'\0{username}\0{password}'
    
    # Base64エンコード
    encoded = base64.b64encode(auth_string.encode()).decode()
    
    return f'AUTH PLAIN {encoded}'

# 使用例
auth_command = smtp_auth_plain('user@example.com', 'password123')
print(auth_command)
# 出力: AUTH PLAIN AHVzZXJAZXhhbXBsZS5jb20AdXNlckBleGFtcGxlLmNvbQBwYXNzd29yZDEyMw==
```

#### LOGIN認証
```python
def smtp_auth_login(username, password):
    """LOGIN認証の実装"""
    steps = [
        'AUTH LOGIN',
        base64.b64encode(username.encode()).decode(),
        base64.b64encode(password.encode()).decode()
    ]
    
    return steps

# 使用例
auth_steps = smtp_auth_login('user@example.com', 'password123')
for step in auth_steps:
    print(step)
# 出力:
# AUTH LOGIN
# dXNlckBleGFtcGxlLmNvbQ==
# cGFzc3dvcmQxMjM=
```

#### CRAM-MD5認証
```python
import hmac
import hashlib
import base64

def smtp_auth_cram_md5(username, password, challenge):
    """CRAM-MD5認証の実装"""
    # チャレンジをデコード
    decoded_challenge = base64.b64decode(challenge)
    
    # HMAC-MD5計算
    digest = hmac.new(
        password.encode(),
        decoded_challenge,
        hashlib.md5
    ).hexdigest()
    
    # レスポンス作成
    response = f'{username} {digest}'
    
    # Base64エンコード
    return base64.b64encode(response.encode()).decode()

# 使用例
challenge = 'PDQxOTI5NDIzNDEuMTI4Mjg0NzJAdm1haWwuZXhhbXBsZS5jb20+'
response = smtp_auth_cram_md5('user@example.com', 'password123', challenge)
print(response)
```

### OAuth 2.0認証（Gmail、Outlook）
```python
import requests
from email.mime.text import MIMEText
import smtplib
import json

class OAuth2SMTP:
    def __init__(self, client_id, client_secret, redirect_uri):
        self.client_id = client_id
        self.client_secret = client_secret
        self.redirect_uri = redirect_uri
        self.access_token = None
        self.refresh_token = None
    
    def get_access_token(self, refresh_token):
        """リフレッシュトークンからアクセストークン取得"""
        token_url = 'https://oauth2.googleapis.com/token'
        
        data = {
            'client_id': self.client_id,
            'client_secret': self.client_secret,
            'refresh_token': refresh_token,
            'grant_type': 'refresh_token'
        }
        
        response = requests.post(token_url, data=data)
        token_info = response.json()
        
        self.access_token = token_info.get('access_token')
        return self.access_token
    
    def send_email_oauth2(self, from_addr, to_addr, subject, body):
        """OAuth2認証でメール送信"""
        if not self.access_token:
            raise Exception("Access token not available")
        
        # Gmail SMTP設定
        smtp_server = 'smtp.gmail.com'
        port = 587
        
        # メール作成
        msg = MIMEText(body)
        msg['Subject'] = subject
        msg['From'] = from_addr
        msg['To'] = to_addr
        
        # SMTP接続
        server = smtplib.SMTP(smtp_server, port)
        server.starttls()
        
        # OAuth2認証
        auth_string = f'user={from_addr}\x01auth=Bearer {self.access_token}\x01\x01'
        server.docmd('AUTH', 'XOAUTH2 ' + base64.b64encode(auth_string.encode()).decode())
        
        # メール送信
        server.send_message(msg)
        server.quit()
```

---

## SMTP over SSL/TLS

### 暗号化の種類

#### SMTPS（暗黙的TLS）
```python
# ポート465での接続例
import smtplib
import ssl

def send_email_smtps(smtp_server, username, password, from_addr, to_addr, subject, body):
    """SMTPS（SSL/TLS暗号化）でメール送信"""
    port = 465
    
    # SSL context作成
    context = ssl.create_default_context()
    
    # SMTP_SSL接続
    with smtplib.SMTP_SSL(smtp_server, port, context=context) as server:
        server.login(username, password)
        
        # メール作成・送信
        message = f"""From: {from_addr}
To: {to_addr}
Subject: {subject}

{body}
"""
        server.sendmail(from_addr, to_addr, message)
```

#### STARTTLS（明示的TLS）
```python
def send_email_starttls(smtp_server, username, password, from_addr, to_addr, subject, body):
    """STARTTLS（明示的TLS）でメール送信"""
    port = 587
    
    with smtplib.SMTP(smtp_server, port) as server:
        # 平文で接続開始
        server.ehlo()
        
        # TLS開始
        server.starttls()
        server.ehlo()  # TLS後に再度EHLO
        
        server.login(username, password)
        
        # メール作成・送信
        message = f"""From: {from_addr}
To: {to_addr}
Subject: {subject}

{body}
"""
        server.sendmail(from_addr, to_addr, message)
```

### TLS設定の検証
```python
import ssl
import socket

class SMTPTLSChecker:
    def __init__(self, hostname, port=587):
        self.hostname = hostname
        self.port = port
    
    def check_tls_support(self):
        """STARTTLS サポートチェック"""
        try:
            with smtplib.SMTP(self.hostname, self.port) as server:
                server.ehlo()
                return server.has_extn('STARTTLS')
        except Exception as e:
            print(f"TLS チェックエラー: {e}")
            return False
    
    def get_certificate_info(self):
        """SSL証明書情報取得"""
        context = ssl.create_default_context()
        
        try:
            with socket.create_connection((self.hostname, 465), timeout=10) as sock:
                with context.wrap_socket(sock, server_hostname=self.hostname) as ssock:
                    cert = ssock.getpeercert()
                    
                    return {
                        'subject': dict(x[0] for x in cert['subject']),
                        'issuer': dict(x[0] for x in cert['issuer']),
                        'version': cert['version'],
                        'serial_number': cert['serialNumber'],
                        'not_before': cert['notBefore'],
                        'not_after': cert['notAfter'],
                        'san': cert.get('subjectAltName', [])
                    }
        except Exception as e:
            print(f"証明書取得エラー: {e}")
            return None
    
    def check_cipher_suites(self):
        """使用可能な暗号スイート確認"""
        context = ssl.create_default_context()
        
        try:
            with socket.create_connection((self.hostname, 465), timeout=10) as sock:
                with context.wrap_socket(sock, server_hostname=self.hostname) as ssock:
                    return {
                        'cipher': ssock.cipher(),
                        'version': ssock.version(),
                        'compression': ssock.compression()
                    }
        except Exception as e:
            print(f"暗号スイート確認エラー: {e}")
            return None

# 使用例
checker = SMTPTLSChecker('smtp.gmail.com')
print("STARTTLS サポート:", checker.check_tls_support())
print("証明書情報:", checker.get_certificate_info())
print("暗号情報:", checker.check_cipher_suites())
```

---

## セキュリティ上の脅威

### 1. オープンリレー

#### 問題の概要
```python
def open_relay_explanation():
    return {
        'definition': '認証なしで任意の送信者から任意の宛先にメール転送',
        'risks': [
            'スパムメール送信の踏み台',
            'フィッシング攻撃の発信源',
            'IPアドレスのブラックリスト登録',
            'サーバーリソースの消費'
        ],
        'test_method': 'telnet経由での不正中継テスト',
        'modern_status': '現在はほぼ全てのサーバーで対策済み'
    }
```

#### オープンリレーテスト
```python
import smtplib
import socket

class OpenRelayTester:
    def __init__(self, target_server, target_port=25):
        self.target_server = target_server
        self.target_port = target_port
    
    def test_open_relay(self):
        """オープンリレーテスト実行"""
        test_cases = [
            {
                'mail_from': 'test@external.com',
                'rcpt_to': 'victim@external.org',
                'description': '外部から外部への中継テスト'
            },
            {
                'mail_from': 'test@external.com',
                'rcpt_to': f'test@{self.target_server}',
                'description': '外部から内部への配送テスト'
            }
        ]
        
        results = []
        
        for test_case in test_cases:
            try:
                result = self._perform_relay_test(
                    test_case['mail_from'],
                    test_case['rcpt_to']
                )
                
                results.append({
                    'test': test_case['description'],
                    'result': result,
                    'vulnerable': result['success']
                })
                
            except Exception as e:
                results.append({
                    'test': test_case['description'],
                    'result': f'Error: {e}',
                    'vulnerable': False
                })
        
        return results
    
    def _perform_relay_test(self, mail_from, rcpt_to):
        """実際の中継テスト実行"""
        try:
            server = smtplib.SMTP(self.target_server, self.target_port, timeout=30)
            server.set_debuglevel(0)
            
            # HELO
            server.helo('relay-test.com')
            
            # MAIL FROM
            mail_response = server.mail(mail_from)
            
            # RCPT TO
            rcpt_response = server.rcpt(rcpt_to)
            
            # QUIT
            server.quit()
            
            return {
                'success': rcpt_response[0] == 250,
                'mail_response': mail_response,
                'rcpt_response': rcpt_response
            }
            
        except smtplib.SMTPRecipientsRefused:
            return {'success': False, 'reason': 'Recipients refused'}
        except smtplib.SMTPSenderRefused:
            return {'success': False, 'reason': 'Sender refused'}
        except Exception as e:
            return {'success': False, 'reason': str(e)}
```

### 2. メールスプーフィング

#### SPF（Sender Policy Framework）
```python
import dns.resolver

class SPFValidator:
    def __init__(self):
        pass
    
    def get_spf_record(self, domain):
        """SPFレコード取得"""
        try:
            answers = dns.resolver.resolve(domain, 'TXT')
            
            for record in answers:
                txt_string = record.to_text().strip('"')
                if txt_string.startswith('v=spf1'):
                    return txt_string
            
            return None
            
        except Exception as e:
            return f"Error: {e}"
    
    def parse_spf_record(self, spf_record):
        """SPFレコード解析"""
        if not spf_record or not spf_record.startswith('v=spf1'):
            return None
        
        mechanisms = []
        qualifiers = {'+': 'Pass', '-': 'Fail', '~': 'SoftFail', '?': 'Neutral'}
        
        parts = spf_record.split()
        
        for part in parts[1:]:  # v=spf1をスキップ
            if part in ['all', '+all', '-all', '~all', '?all']:
                qualifier = qualifiers.get(part[0] if part[0] in qualifiers else '+', 'Pass')
                mechanisms.append({
                    'type': 'all',
                    'qualifier': qualifier,
                    'value': part
                })
            elif part.startswith('ip4:') or part.startswith('ip6:'):
                mechanisms.append({
                    'type': 'ip',
                    'qualifier': qualifiers.get(part[0] if part[0] in qualifiers else '+', 'Pass'),
                    'value': part
                })
            elif part.startswith('include:'):
                mechanisms.append({
                    'type': 'include',
                    'qualifier': qualifiers.get(part[0] if part[0] in qualifiers else '+', 'Pass'),
                    'value': part
                })
            elif part.startswith('mx'):
                mechanisms.append({
                    'type': 'mx',
                    'qualifier': qualifiers.get(part[0] if part[0] in qualifiers else '+', 'Pass'),
                    'value': part
                })
        
        return mechanisms

# 使用例
spf_validator = SPFValidator()
spf_record = spf_validator.get_spf_record('google.com')
print(f"SPF Record: {spf_record}")

parsed = spf_validator.parse_spf_record(spf_record)
for mechanism in parsed:
    print(f"Type: {mechanism['type']}, Qualifier: {mechanism['qualifier']}, Value: {mechanism['value']}")
```

#### DKIM検証
```python
import base64
import hashlib
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import rsa, padding

class DKIMValidator:
    def __init__(self):
        pass
    
    def extract_dkim_signature(self, email_headers):
        """メールヘッダーからDKIM署名抽出"""
        dkim_signature = None
        
        for header in email_headers:
            if header.startswith('DKIM-Signature:'):
                dkim_signature = header[15:].strip()
                break
        
        if not dkim_signature:
            return None
        
        # DKIM署名パラメータ解析
        params = {}
        pairs = dkim_signature.split(';')
        
        for pair in pairs:
            if '=' in pair:
                key, value = pair.split('=', 1)
                params[key.strip()] = value.strip()
        
        return params
    
    def get_dkim_public_key(self, selector, domain):
        """DKIM公開鍵取得"""
        dkim_domain = f"{selector}._domainkey.{domain}"
        
        try:
            answers = dns.resolver.resolve(dkim_domain, 'TXT')
            
            for record in answers:
                txt_string = record.to_text().strip('"')
                if 'k=rsa' in txt_string and 'p=' in txt_string:
                    # 公開鍵抽出
                    parts = txt_string.split(';')
                    for part in parts:
                        if part.strip().startswith('p='):
                            public_key_b64 = part.strip()[2:]
                            return public_key_b64
            
            return None
            
        except Exception as e:
            return f"Error: {e}"
    
    def verify_dkim_signature(self, email_content, dkim_params):
        """DKIM署名検証"""
        try:
            # 署名対象ヘッダー抽出
            signed_headers = dkim_params.get('h', '').split(':')
            
            # 正規化とハッシュ計算
            canonicalized_headers = self._canonicalize_headers(email_content, signed_headers)
            
            # ハッシュ計算
            if dkim_params.get('a', '').startswith('rsa-sha256'):
                hash_func = hashlib.sha256
            else:
                hash_func = hashlib.sha1
            
            header_hash = hash_func(canonicalized_headers.encode()).digest()
            
            # 公開鍵取得
            selector = dkim_params.get('s')
            domain = dkim_params.get('d')
            public_key_b64 = self.get_dkim_public_key(selector, domain)
            
            if not public_key_b64:
                return False
            
            # 署名検証（実装は複雑なため簡略化）
            return True  # 実際の実装では暗号学的検証を行う
            
        except Exception as e:
            print(f"DKIM検証エラー: {e}")
            return False
```

### 3. メール爆弾・DoS攻撃

#### 大量メール送信攻撃
```python
class MailBombProtection:
    def __init__(self):
        self.rate_limits = {
            'per_minute': 60,
            'per_hour': 1000,
            'per_day': 10000
        }
        self.sender_tracking = {}
    
    def check_rate_limit(self, sender_ip, current_time):
        """送信レート制限チェック"""
        if sender_ip not in self.sender_tracking:
            self.sender_tracking[sender_ip] = {
                'minute_count': 0,
                'hour_count': 0,
                'day_count': 0,
                'last_minute': current_time // 60,
                'last_hour': current_time // 3600,
                'last_day': current_time // 86400
            }
        
        tracking = self.sender_tracking[sender_ip]
        current_minute = current_time // 60
        current_hour = current_time // 3600
        current_day = current_time // 86400
        
        # カウンターリセット
        if current_minute > tracking['last_minute']:
            tracking['minute_count'] = 0
            tracking['last_minute'] = current_minute
        
        if current_hour > tracking['last_hour']:
            tracking['hour_count'] = 0
            tracking['last_hour'] = current_hour
        
        if current_day > tracking['last_day']:
            tracking['day_count'] = 0
            tracking['last_day'] = current_day
        
        # レート制限チェック
        if tracking['minute_count'] >= self.rate_limits['per_minute']:
            return False, 'Per-minute limit exceeded'
        
        if tracking['hour_count'] >= self.rate_limits['per_hour']:
            return False, 'Per-hour limit exceeded'
        
        if tracking['day_count'] >= self.rate_limits['per_day']:
            return False, 'Per-day limit exceeded'
        
        # カウンター増加
        tracking['minute_count'] += 1
        tracking['hour_count'] += 1
        tracking['day_count'] += 1
        
        return True, 'OK'
    
    def detect_suspicious_patterns(self, sender_ip, recipients, subject, body):
        """怪しいパターンの検出"""
        suspicious_indicators = []
        
        # 大量の宛先
        if len(recipients) > 100:
            suspicious_indicators.append('Too many recipients')
        
        # 同じ件名の大量送信
        if subject and len(subject) < 10:
            suspicious_indicators.append('Suspiciously short subject')
        
        # 本文が空または短すぎる
        if not body or len(body) < 20:
            suspicious_indicators.append('Empty or very short body')
        
        # 同一IPから短時間での大量アクセス
        tracking = self.sender_tracking.get(sender_ip, {})
        if tracking.get('minute_count', 0) > 30:
            suspicious_indicators.append('High frequency from same IP')
        
        return suspicious_indicators
```

---

## 対策と防御

### 1. Postfix設定例

#### セキュアなPostfix設定
```bash
# /etc/postfix/main.cf

# 基本設定
myhostname = mail.example.com
mydomain = example.com
myorigin = $mydomain
inet_interfaces = all
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain

# セキュリティ設定
# オープンリレー防止
smtpd_relay_restrictions = 
    permit_mynetworks,
    permit_sasl_authenticated,
    defer_unauth_destination

# SMTP認証設定
smtpd_sasl_auth_enable = yes
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_security_options = noanonymous, noplaintext
smtpd_sasl_tls_security_options = noanonymous

# TLS設定
smtpd_tls_cert_file = /etc/ssl/certs/mail.example.com.crt
smtpd_tls_key_file = /etc/ssl/private/mail.example.com.key
smtpd_tls_security_level = may
smtpd_tls_auth_only = yes
smtpd_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
smtpd_tls_ciphers = high
smtpd_tls_exclude_ciphers = aNULL, eNULL, EXPORT, DES, RC4, MD5, PSK, SRP, DSS, AECDH, ADH

# クライアント側TLS
smtp_tls_security_level = may
smtp_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1

# レート制限
smtpd_client_connection_count_limit = 10
smtpd_client_connection_rate_limit = 30
smtpd_client_message_rate_limit = 100

# メッセージサイズ制限
message_size_limit = 52428800  # 50MB

# 送信者検証
smtpd_sender_restrictions = 
    reject_non_fqdn_sender,
    reject_unknown_sender_domain

# 受信者検証
smtpd_recipient_restrictions = 
    reject_non_fqdn_recipient,
    reject_unknown_recipient_domain,
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_unauth_destination

# HELO/EHLO検証
smtpd_helo_restrictions = 
    reject_non_fqdn_helo_hostname,
    reject_invalid_helo_hostname

# データ検証
smtpd_data_restrictions = reject_unauth_pipelining

# ログ設定
maillog_file = /var/log/postfix.log
```

### 2. スパム対策

#### SpamAssassin統合
```bash
# /etc/postfix/master.cf にフィルター追加
smtp      inet  n       -       y       -       -       smtpd
  -o content_filter=spamassassin

spamassassin unix -     n       n       -       -       pipe
  user=spamd argv=/usr/bin/spamc -f -e
  /usr/sbin/sendmail -oi -f ${sender} ${recipient}

# SpamAssassin設定
# /etc/spamassassin/local.cf
required_score 5.0
report_safe 0
rewrite_header Subject [SPAM]
use_bayes 1
bayes_auto_learn 1
```

#### RBL（Realtime Blackhole List）設定
```bash
# /etc/postfix/main.cf
smtpd_client_restrictions = 
    permit_mynetworks,
    reject_rbl_client zen.spamhaus.org,
    reject_rbl_client bl.spamcop.net,
    reject_rbl_client dnsbl.sorbs.net

# DNSBLテスト
dig 127.0.0.2.zen.spamhaus.org
```

### 3. メール認証実装

#### SPF設定
```bash
# DNS TXTレコード例
example.com.    TXT    "v=spf1 mx include:_spf.google.com ip4:203.0.113.0/24 ~all"

# 意味:
# mx: MXレコードのサーバーから送信OK
# include:_spf.google.com: GoogleのSPFを含める
# ip4:203.0.113.0/24: 指定IPレンジから送信OK
# ~all: その他はソフトフェイル
```

#### DKIM設定
```bash
# OpenDKIM設定
# /etc/opendkim/opendkim.conf
Domain                  example.com
Selector                default
KeyFile                 /etc/opendkim/keys/default.private
Socket                  inet:8891@localhost

# 鍵生成
opendkim-genkey -s default -d example.com

# DNS TXTレコード
default._domainkey.example.com.    TXT    "v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA..."
```

#### DMARC設定
```bash
# DNS TXTレコード
_dmarc.example.com.    TXT    "v=DMARC1; p=quarantine; rua=mailto:dmarc-reports@example.com; ruf=mailto:dmarc-forensic@example.com; sp=quarantine; adkim=r; aspf=r"

# パラメータ説明:
# p=quarantine: 認証失敗時は隔離
# rua: 集計レポート送信先
# ruf: フォレンジックレポート送信先
# sp: サブドメインポリシー
# adkim: DKIM alignment mode (relaxed)
# aspf: SPF alignment mode (relaxed)
```

---

## 実際のケーススタディ

### ケース1: 大企業でのメールセキュリティ強化

**状況**: 従業員10,000人の企業でフィッシング攻撃が多発

**実装した対策**:
```yaml
email_security_implementation:
  infrastructure:
    - mail_gateway: "Proofpoint Email Protection"
    - internal_relay: "Postfix with security hardening"
    - authentication: "Active Directory integration"
  
  security_measures:
    spf_record: "v=spf1 include:_spf.proofpoint.com include:spf.protection.outlook.com ~all"
    dkim_selectors: ["selector1", "selector2"]
    dmarc_policy: "p=reject; rua=mailto:dmarc@company.com"
    
  additional_protections:
    - url_rewriting: true
    - attachment_sandboxing: true
    - user_training: "monthly phishing simulations"
    - incident_response: "automated quarantine and analysis"
```

**結果**:
- フィッシング検出率: 99.7%
- 偽陽性率: 0.1%未満
- セキュリティインシデント: 90%減少

### ケース2: ISPでのスパム対策

**課題**: 顧客からのスパム送信とオープンリレー悪用

**対策実装**:
```python
class ISPSpamProtection:
    def __init__(self):
        self.protection_layers = {
            'connection_level': [
                'IP reputation checking',
                'Rate limiting per IP',
                'Greylisting implementation'
            ],
            'content_level': [
                'SpamAssassin integration',
                'Custom rule development',
                'Machine learning classification'
            ],
            'authentication_level': [
                'Mandatory SMTP AUTH',
                'SPF/DKIM/DMARC validation',
                'Customer domain verification'
            ],
            'monitoring_level': [
                'Real-time traffic analysis',
                'Abuse detection algorithms',
                'Customer notification system'
            ]
        }
    
    def implement_customer_protection(self):
        return {
            'outbound_scanning': 'All customer emails scanned',
            'reputation_monitoring': 'IP reputation tracking',
            'automatic_blocking': 'Compromised accounts auto-suspended',
            'customer_education': 'Security best practices training'
        }
```

### ケース3: 中小企業でのクラウドメール移行

**シナリオ**: オンプレミスからOffice 365への移行

**移行計画**:
```python
class CloudMigrationPlan:
    def __init__(self):
        self.migration_phases = [
            {
                'phase': 1,
                'description': 'DNS preparation and validation',
                'tasks': [
                    'SPF record update',
                    'DKIM configuration',
                    'DMARC policy setup',
                    'MX record preparation'
                ]
            },
            {
                'phase': 2,
                'description': 'Pilot user migration',
                'tasks': [
                    'Select 10% of users',
                    'Mailbox migration',
                    'Client configuration',
                    'Testing and validation'
                ]
            },
            {
                'phase': 3,
                'description': 'Full migration',
                'tasks': [
                    'Batch user migration',
                    'DNS cutover',
                    'Decommission old server',
                    'Post-migration support'
                ]
            }
        ]
    
    def get_security_improvements(self):
        return {
            'before_migration': [
                'Basic spam filtering',
                'No encryption in transit',
                'Limited authentication options',
                'Manual security updates'
            ],
            'after_migration': [
                'Advanced Threat Protection',
                'Automatic encryption',
                'Multi-factor authentication',
                'Cloud-based security updates',
                'Data Loss Prevention'
            ]
        }
```

**成果**:
- セキュリティレベル向上: 大幅改善
- 運用コスト削減: 60%削減
- ユーザー満足度: 95%
- ダウンタイム: 2時間未満

---

## まとめ

### SMTPの重要ポイント

1. **基盤プロトコル**: インターネットメールの基盤となる重要な技術
2. **セキュリティの進化**: 認証、暗号化、送信者検証の重要性
3. **継続的な対策**: スパム、フィッシング等の新しい脅威への対応
4. **標準化**: RFC準拠による相互運用性の確保

### 管理者への推奨事項

- **認証の強制**: SMTP AUTHの必須化
- **暗号化の実装**: STARTTLS/SMTPSの使用
- **送信者検証**: SPF/DKIM/DMARCの実装
- **継続的監視**: ログ分析とセキュリティ監視

### 今後の展望

- **BIMI（Brand Indicators for Message Identification）**: ブランド表示の標準化
- **MTA-STS（Mail Transfer Agent Strict Transport Security）**: 強制暗号化
- **ARC（Authenticated Received Chain）**: 転送メールの認証チェーン
- **AI/ML活用**: より高度なスパム・フィッシング検出

SMTPは長い歴史を持つプロトコルですが、セキュリティ要件の高まりとともに継続的に進化しています。適切な理解と実装により、安全で信頼性の高いメール通信システムを構築できます。

---

**参考資料**:
- RFC 5321: Simple Mail Transfer Protocol
- RFC 5322: Internet Message Format  
- RFC 7208: Sender Policy Framework (SPF)
- RFC 6376: DomainKeys Identified Mail (DKIM)
- RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)

