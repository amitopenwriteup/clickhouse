# ClickHouse Backup Setup — Hinglish mein समझें

चलो, इस guide को Hinglish में समझते हैं — ClickHouse में backup disk setup करने का पूरा process, disk बनाने से लेकर restore तक।

## 1. Backup Directory Setup

```
$ sudo mkdir -p /var/lib/clickhouse/backups
$ sudo chown -R clickhouse:clickhouse /var/lib/clickhouse/backups
```

ये disk पे वो physical location है जहाँ ClickHouse backup archives को store करेगा। इस directory का owner `clickhouse` user होना चाहिए (या जो भी user server run करता है) — **वरना writes permission error के साथ fail हो जाएंगे**।

## 2. Config File Create करना

```
$ sudo nano /etc/clickhouse-server/config.d/backup_disk.xml
```

`config.xml` को directly edit करने से better है `config.d/` के अंदर नया file बनाना। ClickHouse startup के time इस folder के सारे files को automatically merge कर लेता है, इसलिए तुम्हारा original config clean रहता है और future में rollback भी आसान होता है।

## 3. Disk Register करना (`storage_configuration` के अंदर)

`backup_disk.xml` में ये paste करो:

```xml
<clickhouse>
    <storage_configuration>
        <disks>
            <backups>
                <type>local</type>
                <path>/var/lib/clickhouse/backups/</path>
            </backups>
        </disks>
    </storage_configuration>

    <backups>
        <allowed_disk>backups</allowed_disk>
        <allowed_path>/var/lib/clickhouse/backups/</allowed_path>
    </backups>
</clickhouse>
```

`<disks><backups>` block एक disk define करता है जिसका नाम "backups" है, और ये directory 1 में बनाई हुई directory को point करता है। `<backups><allowed_disk>` block modern ClickHouse में जरूरी है — बिना disk नाम को explicitly allowlist किए, `BACKUP ... TO Disk('backups', ...)` reject हो जाएगा, भले ही disk exist करती हो।

## 4. XML Validate करना और Server Restart करना

```
$ sudo clickhouse extract-from-config --config-file /etc/clickhouse-server/config.xml --key=backups
$ sudo service clickhouse-server restart
```

`extract-from-config` एक quick sanity check है ये confirm करने के लिए कि तुम्हारा XML well-formed है, restart करने से पहले — malformed config की वजह से server start ही नहीं हो पाएगा।

## 5. Disk Visibility Confirm करना

```sql
SELECT name, path, type FROM system.disks WHERE name = 'backups';
```

तुम्हें वो path वापस दिखना चाहिए जो तुमने configure किया था। अगर table खाली है, तो मतलब config pick up नहीं हुआ — double check करो कि file `config.d/` में है और server वाकई restart हुआ (`SELECT uptime();` से confirm करो)।

## 6. Test Backup लेना

```sql
BACKUP TABLE sales_db.orders TO Disk('backups', 'test_backup.zip');
```

अगर ये succeed हो जाता है, तो backup disk पूरी तरह wired up है और restore/incremental backups जैसे बाकी steps के लिए ready है।

> **Note:** ClickHouse Cloud पर तुम इस तरह disks manage नहीं करते — backups platform handle करता है, और ये local-disk setup सिर्फ self-managed/on-prem clusters पर लागू होता है। Production के लिए ज़्यादातर teams disk को S3 पर point करती हैं (`<type>s3</type>` endpoint/credentials के साथ), ताकि node खोने पर भी backups सुरक्षित रहें।

## 7-8. Table और Database का Full Backup

```sql
BACKUP TABLE sales_db.orders TO Disk('backups', 'orders_full_2024.zip');
BACKUP DATABASE sales_db TO Disk('backups', 'sales_db_full.zip');
```

पहला command `orders` table का self-contained backup archive बनाता है। दूसरा पूरे schema और data के disaster-recovery के लिए useful है।

## 9-11. Data Loss Simulate करके Restore Verify करना

```sql
TRUNCATE TABLE sales_db.orders;
RESTORE TABLE sales_db.orders FROM Disk('backups', 'orders_full_2024.zip');
SELECT count() FROM sales_db.orders;
```

पहले table को empty करो ताकि verify कर सको कि restore वाकई काम करता है। `RESTORE` command table को फिर से बनाता है और archive से उसका data reload करता है। आखिर में row count check करके confirm करो कि restore complete हुआ और कोई data loss नहीं हुआ।

## 12. Incremental Backup (Optional)

```sql
BACKUP TABLE sales_db.orders TO Disk('backups', 'orders_incr.zip')
  SETTINGS base_backup = Disk('backups', 'orders_full_2024.zip');
```

ये सिर्फ base backup के बाद हुए changes को store करता है, जिससे time और space दोनों बचते हैं।
