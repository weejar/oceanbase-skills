# 数据加密（TDE/传输加密/列加密）

> 适用版本：OceanBase V4.2+
> 兼容模式：MySQL Mode / Oracle Mode

---

## 概述

OceanBase 提供多层加密体系：

| 加密层 | 方式 | 保护范围 | 配置复杂度 |
|--------|------|---------|-----------|
| 传输加密 | SSL/TLS | 客户端↔OBServer | 低 |
| 存储加密 | TDE | 数据文件、日志文件 | 中 |
| 列加密 | 函数加密 | 单个敏感列 | 低 |
| 密钥管理 | 内部/外部KMS | 加密密钥 | 中-高 |

---

## 1. 传输加密（SSL/TLS）

### 1.1 开启 SSL

```bash
# 1. 生成 CA 证书和密钥
openssl genrsa 2048 > ca-key.pem
openssl req -new -x509 -nodes -days 3650 \
  -key ca-key.pem > ca-cert.pem

# 2. 生成 Server 证书
openssl req -newkey rsa:2048 -nodes -days 3650 \
  -keyout server-key.pem -out server-req.pem
openssl x509 -req -in server-req.pem -days 3650 \
  -CA ca-cert.pem -CAkey ca-key.pem -set_serial 01 \
  -out server-cert.pem

# 3. 生成 Client 证书
openssl req -newkey rsa:2048 -nodes -days 3650 \
  -keyout client-key.pem -out client-req.pem
openssl x509 -req -in client-req.pem -days 3650 \
  -CA ca-cert.pem -CAkey ca-key.pem -set_serial 01 \
  -out client-cert.pem
```

### 1.2 配置 OBServer SSL

```sql
-- sys 租户执行
ALTER SYSTEM SET ssl_external_kms_info = '{
  "ca_cert": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----",
  "server_cert": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----",
  "server_key": "-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----"
}';

-- 等待 SSL 生效
ALTER SYSTEM SET enable_ssl = TRUE;
```

### 1.3 客户端 SSL 连接

**JDBC（MySQL Mode）：**
```java
String url = "jdbc:mysql://obhost:2883/db?" +
    "useSSL=true" +
    "&verifyServerCertificate=true" +
    "&trustCertificateKeyStoreUrl=file:///path/to/truststore.jks" +
    "&trustCertificateKeyStorePassword=changeit";
```

**JDBC（Oracle Mode）：**
```java
String url = "jdbc:oceanbase://obhost:2883/service?" +
    "ssl=true" +
    "&trustServerCertificate=false" +
    "&trustStore=file:///path/to/truststore.jks" +
    "&trustStorePassword=changeit";
```

**MySQL 客户端：**
```bash
mysql -h obhost -P 2883 -u root -p --ssl-ca=ca-cert.pem --ssl-cert=client-cert.pem --ssl-key=client-key.pem
```

### 1.4 OBProxy SSL 终结

```bash
# obproxy 配置 SSL
obproxy --ssl-cert=server-cert.pem --ssl-key=server-key.pem --ssl-ca=ca-cert.pem
```

---

## 2. 透明数据加密（TDE）

### 2.1 内部密钥管理

```sql
-- 开启 TDE（sys 租户）
ALTER SYSTEM SET encryption_algorithm = 'AES_256';
ALTER SYSTEM SET enable_tde = TRUE;

-- 查看加密状态
SELECT * FROM oceanbase.__all_virtual_tenant_info
WHERE encryption_status != 0;
```

### 2.2 外部密钥管理（KMS）

支持对接外部 KMS（如阿里云 KMS、AWS KMS、HashiCorp Vault）：

```sql
-- 配置外部 KMS
ALTER SYSTEM SET external_kms_info = '{
  "kms_type": "aliyun",
  "kms_region": "cn-hangzhou",
  "kms_instance_id": "kms-cn-xxxx",
  "access_key_id": "LTAI...",
  "access_key_secret": "..."
}';

-- 使用 KMS 密钥
ALTER SYSTEM SET master_key_id = 'cmk-xxxx';
ALTER SYSTEM SET enable_tde = TRUE;
```

### 2.3 密钥轮转

```sql
-- 查看当前主密钥
SELECT key_id, status, create_time
FROM oceanbase.__all_virtual_encryption_key_info;

-- 轮转密钥（生成新密钥，后台异步重加密）
ALTER SYSTEM ROTATE ENCRYPTION KEY;

-- 轮转状态检查
SELECT * FROM oceanbase.__all_virtual_encryption_rotate_task;
```

---

## 3. 列加密

### 3.1 使用内置加密函数

```sql
-- 创建加密表（Oracle Mode）
CREATE TABLE customer_sensitive (
  customer_id  NUMBER PRIMARY KEY,
  name         VARCHAR2(100),
  id_card      VARCHAR2(256),  -- 加密存储
  phone        VARCHAR2(256),  -- 加密存储
  email        VARCHAR2(200)
);

-- 插入加密数据
INSERT INTO customer_sensitive VALUES (
  1, '张三',
  DBMS_CRYPTO.ENCRYPT(
    UTL_I18N.STRING_TO_RAW('110101199001011234', 'AL32UTF8'),
    DBMS_CRYPTO.AES256_CBC + DBMS_CRYPTO.CHAIN_CBC + DBMS_CRYPTO.PAD_PKCS5,
    UTL_I18N.STRING_TO_RAW('MySecretKey123456', 'AL32UTF8')
  ),
  DBMS_CRYPTO.ENCRYPT(
    UTL_I18N.STRING_TO_RAW('13800138000', 'AL32UTF8'),
    DBMS_CRYPTO.AES256_CBC + DBMS_CRYPTO.CHAIN_CBC + DBMS_CRYPTO.PAD_PKCS5,
    UTL_I18N.STRING_TO_RAW('MySecretKey123456', 'AL32UTF8')
  ),
  'zhangsan@example.com'
);

-- 解密查询
SELECT
  customer_id,
  name,
  UTL_I18N.RAW_TO_CHAR(
    DBMS_CRYPTO.DECRYPT(
      id_card,
      DBMS_CRYPTO.AES256_CBC + DBMS_CRYPTO.CHAIN_CBC + DBMS_CRYPTO.PAD_PKCS5,
      UTL_I18N.STRING_TO_RAW('MySecretKey123456', 'AL32UTF8')
    ),
    'AL32UTF8'
  ) AS id_card_decrypted
FROM customer_sensitive;
```

### 3.2 MySQL Mode 列加密

```sql
-- 使用 AES 函数
INSERT INTO customer_sensitive VALUES (
  1, '张三',
  HEX(AES_ENCRYPT('110101199001011234', 'MySecretKey123456')),
  HEX(AES_ENCRYPT('13800138000', 'MySecretKey123456')),
  'zhangsan@example.com'
);

-- 解密查询
SELECT
  customer_id,
  name,
  CAST(AES_DECRYPT(UNHEX(id_card), 'MySecretKey123456') AS CHAR) AS id_card,
  CAST(AES_DECRYPT(UNHEX(phone), 'MySecretKey123456') AS CHAR) AS phone
FROM customer_sensitive;
```

### 3.3 应用层加密

推荐在应用层加密后存储，避免密钥泄露到数据库：

```java
// 使用 HSM 或 KMS 进行应用层加密
public class FieldEncryptor {
    private final SecretKey key;

    public String encrypt(String plaintext) {
        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        cipher.init(Cipher.ENCRYPT_MODE, key);
        byte[] iv = new byte[12]; // 随机 IV
        SecureRandom random = new SecureRandom();
        random.nextBytes(iv);
        cipher.updateAAD(iv);
        byte[] encrypted = cipher.doFinal(plaintext.getBytes(UTF_8));
        // 存储: base64(iv + encrypted)
        return Base64.getEncoder().encodeToString(concat(iv, encrypted));
    }
}
```

---

## 4. 密钥管理最佳实践

### 密钥层级

```
MEK (Master Encryption Key)
  └── DEK (Data Encryption Key) — 每个租户独立
        └── 数据加密
```

### 密钥管理规则

| 规则 | 说明 |
|------|------|
| 密钥分离 | 不同租户使用不同 DEK |
| 定期轮转 | MEK 每 90 天轮转，DEK 每 180 天 |
| 访问控制 | 密钥访问权限最小化 |
| 备份恢复 | 密钥独立备份到安全位置 |
| 删除保护 | 删除密钥前确认无数据依赖 |

---

## 5. 安全配置检查清单

```sql
-- 检查 SSL 状态
SHOW VARIABLES LIKE '%ssl%';
-- Oracle Mode:
SELECT * FROM v$parameter WHERE name LIKE '%ssl%';

-- 检查 TDE 状态
SELECT enable_tde, encryption_algorithm FROM oceanbase.__all_virtual_tenant_info;

-- 检查是否有未加密的敏感表
SELECT table_name, column_name
FROM information_schema.columns
WHERE table_schema = 'app_db'
  AND column_name IN ('password', 'id_card', 'phone', 'bank_card', 'salary')
  AND data_type NOT LIKE '%encrypted%';
```

---

## 常见错误

| 问题 | 原因 | 解决 |
|------|------|------|
| SSL 握手失败 | 证书链不完整 | 确保证书链顺序正确 |
| TDE 开启失败 | 无有效 KMS 配置 | 检查外部 KMS 连通性 |
| 密钥轮转卡住 | 大量数据重加密中 | 等待后台任务完成 |
| 解密数据乱码 | 编码不一致 | 统一使用 UTF-8 |
| 列加密后无法索引 | 加密数据不可比较 | 对确定性加密值建函数索引 |

---

## 参考文档

- SSL 配置：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/configure-ssl
- TDE 透明加密：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/transparent-data-encryption
- 外部密钥管理：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/configure-external-kms
- DBMS_CRYPTO 包：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/oracle-mode/dbms_crypto
