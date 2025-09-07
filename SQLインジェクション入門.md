# SQLインジェクション入門

## 目次
1. [SQLインジェクションとは](#sqlインジェクションとは)
2. [SQLインジェクションの仕組み](#sqlインジェクションの仕組み)
3. [SQLインジェクションの種類](#sqlインジェクションの種類)
4. [攻撃例と実演](#攻撃例と実演)
5. [被害の範囲](#被害の範囲)
6. [検出方法](#検出方法)
7. [対策方法](#対策方法)
8. [予防策](#予防策)
9. [実際のケーススタディ](#実際のケーススタディ)
10. [まとめ](#まとめ)

---

## SQLインジェクションとは

**SQLインジェクション（SQL Injection）** は、Webアプリケーションの脆弱性を悪用した攻撃手法の一つです。攻撃者がWebフォームやURLパラメータなどの入力フィールドに悪意のあるSQLコードを挿入（インジェクト）することで、データベースに対して意図しない操作を実行させる攻撃です。

### 基本概念
- **SQL**: データベースを操作するための言語
- **インジェクション**: 悪意のあるコードを注入すること
- **脆弱性**: アプリケーションのセキュリティ上の欠陥

---

## SQLインジェクションの仕組み

### 正常な処理の流れ
1. ユーザーがWebフォームにデータを入力
2. アプリケーションがSQL文を構築
3. データベースがSQL文を実行
4. 結果をユーザーに返す

### 攻撃時の流れ
1. 攻撃者が入力フィールドに悪意のあるSQLコードを入力
2. アプリケーションが攻撃者の入力を含むSQL文を構築
3. データベースが意図しないSQL文を実行
4. 機密情報の漏洩や不正操作が発生

### 脆弱性が発生する原因
- **入力値の検証不足**: ユーザーの入力を適切にチェックしていない
- **SQLクエリの動的生成**: 入力値を直接SQL文に組み込んでいる
- **エスケープ処理の不備**: 特殊文字の適切な処理ができていない

---

## SQLインジェクションの種類

### 1. Classic SQLインジェクション（In-band）
最も一般的なタイプで、攻撃結果が直接画面に表示される

#### Union-based SQLインジェクション
```sql
-- 正常なクエリ
SELECT * FROM users WHERE id = 1;

-- 攻撃クエリ
SELECT * FROM users WHERE id = 1 UNION SELECT username, password FROM admin_users;
```

#### Error-based SQLインジェクション
```sql
-- エラーメッセージを利用した攻撃
SELECT * FROM users WHERE id = 1 AND (SELECT COUNT(*) FROM information_schema.tables) > 0;
```

### 2. Blind SQLインジェクション（Inferential）
攻撃結果が直接表示されないが、アプリケーションの動作から情報を推測

#### Boolean-based Blind SQLインジェクション
```sql
-- 条件によってページの表示が変わることを利用
SELECT * FROM users WHERE id = 1 AND 1=1; -- True（正常表示）
SELECT * FROM users WHERE id = 1 AND 1=2; -- False（異なる表示）
```

#### Time-based Blind SQLインジェクション
```sql
-- 応答時間の違いを利用
SELECT * FROM users WHERE id = 1 AND SLEEP(5); -- 5秒の遅延で判断
```

### 3. Out-of-band SQLインジェクション
別の通信チャネル（DNS、HTTPなど）を使用してデータを取得

---

## 攻撃例と実演

### 基本的な攻撃例

#### ログインフォームへの攻撃
```php
// 脆弱なPHPコード例
$username = $_POST['username'];
$password = $_POST['password'];
$query = "SELECT * FROM users WHERE username = '$username' AND password = '$password'";
```

**攻撃入力例:**
- Username: `admin' --`
- Password: （任意）

**生成されるSQL:**
```sql
SELECT * FROM users WHERE username = 'admin' --' AND password = 'password'
```
`--` はSQLのコメントなので、パスワードチェックが無効化される

#### データ抽出攻撃
**攻撃入力:**
```
1' UNION SELECT table_name, column_name FROM information_schema.columns --
```

**生成されるSQL:**
```sql
SELECT * FROM products WHERE id = '1' UNION SELECT table_name, column_name FROM information_schema.columns --'
```

### 高度な攻撃例

#### データベース情報の取得
```sql
-- データベース名の取得
1' UNION SELECT database(), version() --

-- テーブル一覧の取得
1' UNION SELECT table_name, null FROM information_schema.tables WHERE table_schema = database() --

-- カラム情報の取得
1' UNION SELECT column_name, data_type FROM information_schema.columns WHERE table_name = 'users' --
```

---

## 被害の範囲

### 1. データの機密性への影響
- **個人情報の漏洩**: 氏名、住所、電話番号、メールアドレス
- **認証情報の窃取**: ユーザー名、パスワード、APIキー
- **機密データの取得**: 財務情報、営業秘密、顧客データ

### 2. データの完全性への影響
- **データの改ざん**: 既存レコードの不正な変更
- **データの削除**: 重要な情報の消失
- **不正データの挿入**: 偽の情報の追加

### 3. データの可用性への影響
- **サービス停止**: データベースサーバーのクラッシュ
- **パフォーマンス低下**: 大量のクエリによる負荷
- **システム全体の障害**: 連鎖的な影響

### 4. その他の影響
- **法的責任**: 個人情報保護法違反
- **経済的損失**: 復旧費用、賠償金、信頼失墜
- **レピュテーション損害**: 企業イメージの悪化

---

## 検出方法

### 1. 手動テスト
#### 基本的なテストケース
```
' (シングルクォート)
" (ダブルクォート)
; (セミコロン)
-- (コメント)
/* */ (ブロックコメント)
```

#### エラーベースの検出
```
' OR 1=1 --
' AND 1=2 --
' UNION SELECT null --
```

### 2. 自動化ツール
#### オープンソースツール
- **SQLMap**: 最も有名なSQLインジェクション検出ツール
- **jSQL Injection**: Java製のGUIツール
- **NoSQLMap**: NoSQLデータベース向け

#### 商用ツール
- **Burp Suite Professional**: Webアプリケーション脆弱性スキャナー
- **OWASP ZAP**: オープンソースの脆弱性スキャナー
- **Nessus**: 包括的な脆弱性スキャナー

### 3. ログ監視
#### 監視すべきパターン
```
SELECT.*FROM.*information_schema
UNION.*SELECT
OR.*1=1
SLEEP\([0-9]+\)
WAITFOR.*DELAY
```

---

## 対策方法

### 1. プリペアドステートメント（推奨）
```php
// PHP PDO例
$stmt = $pdo->prepare("SELECT * FROM users WHERE username = ? AND password = ?");
$stmt->execute([$username, $password]);
```

```java
// Java JDBC例
String sql = "SELECT * FROM users WHERE username = ? AND password = ?";
PreparedStatement stmt = connection.prepareStatement(sql);
stmt.setString(1, username);
stmt.setString(2, password);
```

```python
# Python例
cursor.execute("SELECT * FROM users WHERE username = %s AND password = %s", 
               (username, password))
```

### 2. 入力値の検証とサニタイゼーション
```php
// 入力値の検証
function validateInput($input) {
    // 長さチェック
    if (strlen($input) > 50) {
        return false;
    }
    
    // 文字種チェック（英数字のみ許可）
    if (!preg_match('/^[a-zA-Z0-9]+$/', $input)) {
        return false;
    }
    
    return true;
}

// エスケープ処理
$username = mysqli_real_escape_string($connection, $_POST['username']);
```

### 3. 最小権限の原則
```sql
-- アプリケーション用のデータベースユーザーを作成
CREATE USER 'webapp_user'@'localhost' IDENTIFIED BY 'strong_password';

-- 必要最小限の権限のみ付与
GRANT SELECT, INSERT, UPDATE ON app_database.users TO 'webapp_user'@'localhost';
GRANT SELECT ON app_database.products TO 'webapp_user'@'localhost';

-- 危険な権限は付与しない（DROP, CREATE, ALTER等）
```

### 4. エラーハンドリング
```php
// 詳細なエラー情報を表示しない
try {
    $result = $stmt->execute();
} catch (PDOException $e) {
    // ログに記録
    error_log("Database error: " . $e->getMessage());
    
    // ユーザーには汎用メッセージを表示
    die("システムエラーが発生しました。管理者にお問い合わせください。");
}
```

---

## 予防策

### 1. セキュアコーディング標準
#### 開発ガイドライン
- **OWASP Top 10**: 最新の脅威情報を参照
- **セキュアコーディング規約**: チーム内での統一ルール
- **コードレビュー**: 複数人でのセキュリティチェック

#### 使用禁止事項
```php
// 使用禁止: 動的SQL文の構築
$query = "SELECT * FROM users WHERE id = " . $_GET['id'];

// 使用禁止: 文字列連結によるクエリ作成
$query = "SELECT * FROM users WHERE name = '" . $username . "'";
```

### 2. WAF（Web Application Firewall）
#### 設定例
```apache
# mod_security設定例
SecRule ARGS "@detectSQLi" \
    "id:1001,\
    phase:2,\
    block,\
    msg:'SQL Injection Attack Detected',\
    logdata:'Matched Data: %{MATCHED_VAR} found within %{MATCHED_VAR_NAME}'"
```

### 3. 定期的なセキュリティ監査
#### 実施項目
- **脆弱性スキャン**: 自動化ツールによる定期チェック
- **ペネトレーションテスト**: 専門家による侵入テスト
- **コード監査**: ソースコードの静的解析

---

## 実際のケーススタディ

### ケース1: ECサイトでの顧客情報漏洩
**概要**: オンラインショップの商品検索機能にSQLインジェクション脆弱性

**攻撃手法**:
```
https://example-shop.com/search?q=laptop' UNION SELECT customer_name, credit_card FROM customers --
```

**被害**:
- 10万人分の顧客情報漏洩
- クレジットカード情報の流出
- 損害額: 約5億円

**対策後**:
- プリペアドステートメントの導入
- 入力値検証の強化
- WAFの導入

### ケース2: 金融機関での不正送金
**概要**: インターネットバンキングシステムの脆弱性を悪用

**攻撃手法**:
```sql
-- 残高チェックをバイパス
UPDATE accounts SET balance = balance - 1000000 WHERE account_id = 'victim' AND balance >= 1000000 OR 1=1 --
```

**被害**:
- 複数口座からの不正送金
- 総額3億円の被害
- システム停止による業務影響

**対策後**:
- トランザクション処理の見直し
- 多要素認証の導入
- リアルタイム監視システムの構築

---

## まとめ

### SQLインジェクションの重要ポイント
1. **深刻な脅威**: データベース全体が危険にさらされる可能性
2. **対策は必須**: すべてのWebアプリケーション開発者が理解すべき
3. **予防が重要**: 事後対応よりも事前の対策が効果的

### 開発者への推奨事項
- **プリペアドステートメントの使用**: 最も効果的な対策
- **入力値検証の実装**: 多層防御の一環として
- **セキュリティ教育の継続**: 最新の脅威情報の把握
- **定期的な監査**: 脆弱性の早期発見

### 組織への推奨事項
- **セキュリティポリシーの策定**: 明確なガイドラインの設定
- **開発プロセスの見直し**: セキュリティを考慮した開発手法
- **インシデント対応計画**: 被害を最小限に抑える準備
- **継続的な改善**: 脅威の変化に対応した対策の更新

SQLインジェクションは古くから知られている攻撃手法ですが、現在でも多くのWebアプリケーションで発見される深刻な脆弱性です。適切な知識と対策により、この脅威から組織とユーザーを守ることができます。

---

**参考資料**:
- OWASP SQL Injection Prevention Cheat Sheet
- CWE-89: Improper Neutralization of Special Elements used in an SQL Command
- NIST SP 800-53: Security and Privacy Controls for Federal Information Systems
