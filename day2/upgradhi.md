Theek hai, ab main aisi style mein likhta hoon jahan **हिंदी शब्द देवनागरी में** होंगे और English words/technical terms Roman script mein रहेंगे:

## थ्योरी वाला भाग (1.3) — पहले ये समझो

**Changelog पहले चेक करो:** ClickHouse बहुत frequently release निकालता है। तो जब भी upgrade करो, especially major version jump (जैसे 23.x से 24.x), पहले changelog ज़रूर पढ़ो — कहीं breaking changes तो नहीं हैं।

**Staging पहले, production बाद में:** Direct production में upgrade मत मारो। पहले staging/test environment में try करो। फिर production में भी **rolling upgrade** करो — मतलब एक-एक replica/node upgrade करो, सब एक साथ नहीं। इससे अगर कुछ गलत हुआ तो सिर्फ एक node affect होगा, पूरा cluster नहीं गिरेगा।

**Zero downtime possible है:** Replicated cluster में ये इसलिए possible है क्योंकि replicas थोड़ी देर के लिए अलग-अलग minor version पे चल सकते हैं बिना problem के।

**Backup लेना मत भूलना:** Major version upgrade से पहले backup ज़रूर लो — अगर कुछ बिगड़ जाए तो rollback कर सको।

**Rocky Linux का context:** ये lab Rocky Linux (RHEL family) पे based है, इसलिए:
- Package manage करने के लिए `dnf` या `yum` use होता है (Ubuntu वाला `apt-get` नहीं)
- Services को `systemctl` से manage करते हैं (Debian/Ubuntu वाला पुराना `service` command नहीं)

---

## Hands-On Steps (Rocky Linux / RHEL-family)

**Step 1: Current version check करो**
```sql
SELECT version();
```
ये तुम्हारा baseline set करता है — upgrade से पहले पता होना चाहिए कि अभी कौन सी version चल रही है।

**Step 2: Target version के release notes/changelog पढ़ो**
इससे पता चलता है कि कोई breaking changes हैं, कोई settings deprecate हुई हैं, या कोई new defaults आए हैं।

**Step 3: Critical databases का backup लो** (Lab 1.2 देखो)
अगर upgrade में कुछ issue आए तो rollback path मिल जाएगा।

**Step 4: (सिर्फ पहली बार) ClickHouse का RPM repository add करो**
```bash
$ sudo yum install -y yum-utils
$ sudo yum-config-manager --add-repo https://packages.clickhouse.com/rpm/clickhouse.repo
```
ये ClickHouse का official RPM repo को dnf/yum के साथ register करता है, ताकि package installs और upgrades यहीं से आएं। अगर Day 1 install के दौरान repo पहले से configure हो चुका है, तो ये step skip कर दो।

**Step 5: Repository cache update करो और नई version install करो**
```bash
$ sudo dnf clean all
$ sudo dnf update -y clickhouse-server clickhouse-client
```
ये ClickHouse repo से नए RPM packages pull और install करता है। अगर packages अभी installed ही नहीं हैं, तो इसकी जगह ये चलाओ: `dnf install -y clickhouse-server clickhouse-client`

**Step 6: ClickHouse server service restart करो**
```bash
$ sudo systemctl restart clickhouse-server
```
Rocky Linux systemd use करता है, इसलिए service manage करने के लिए `systemctl` standard तरीका है। (पुराना `service clickhouse-server restart` syntax भी काम करता है एक compatibility wrapper के through, लेकिन `systemctl` ही preferred है।)

**Step 7: Confirm करो कि service active और enabled है**
```bash
$ sudo systemctl status clickhouse-server
$ sudo systemctl is-enabled clickhouse-server
```
ये confirm करता है कि upgraded service cleanly start हुई है और reboot के बाद भी automatically आएगी।

**Step 8: नई version confirm करो और logs में errors check करो**
```sql
SELECT version();
```
```bash
$ sudo tail -n 100 /var/log/clickhouse-server/clickhouse-server.log
```
ये verify करता है कि upgrade successful रहा और startup clean था।

**Step 9: Key tables पर smoke-test queries चलाओ**
```sql
SELECT count() FROM sales_db.orders;
```
ये confirm करता है कि data और query behavior upgrade के बाद भी affected नहीं हुए।

**Step 10: (सिर्फ cluster के लिए) Steps 5-9 को एक-एक replica/node पर repeat करो**
Rolling upgrades से पूरे cluster का downtime avoid होता है, और तुम एक node पर issues catch कर सकते हो इससे पहले कि वो पूरे cluster को affect करें।

> **NOTE:** अगर तुम्हारे Rocky nodes पर SELinux enforcing mode में है, तो `sudo sestatus` check करो और upgrade के बाद `/var/log/audit/audit.log` में clickhouse-server से related कोई denials तो नहीं आए, वो review करो। RHEL-family systems पर RPM installs, Debian/Ubuntu installs की तुलना में SELinux policy से ज़्यादा commonly affect होते हैं।

---

कोई particular step पे doubt है या practically चलाते वक्त कोई error आया? बताओ, उसी हिसाब से आगे मदद करता हूं।
