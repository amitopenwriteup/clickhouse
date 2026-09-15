# ClickHouse ODBC Driver — Hinglish समझाइश

## Slide 1-2: ODBC Driver क्या है

**ODBC Driver** एक standard interface है जो ODBC-aware applications (जैसे Power BI, Excel, या कोई भी scripting environment) को ClickHouse से connect करने देता है — बिना ClickHouse का अपना specific protocol जाने।

- **Standards-based access:** ODBC API implement करता है, इसलिए कोई भी ODBC-compatible tool ClickHouse को query कर सकता है।
- **HTTP under the hood:** Driver असल में HTTP protocol use करता है ClickHouse server से बात करने के लिए — ये protocol हर deployment (local, cloud) में support होता है।
- **Works everywhere:** Local install हो, cloud-managed service हो, या HTTP-only environment हो — सबमें consistently काम करता है।
- **Open source:** GitHub पे `ClickHouse-ODBC` repo में maintain होता है, active development चल रही है।

---

## Slide 3: ये Connect कैसे होता है

**Flow ये है:**

```
ODBC-Compatible App (जैसे Power BI) → ClickHouse ODBC Driver → ClickHouse Server (HTTP protocol से)
```

Application SQL query भेजता है, driver उसे translate करके एक HTTP request बनाता है जो ClickHouse समझता है, और result वापिस ODBC result set के form में आता है। यही वजह है कि *same driver* local install और ClickHouse Cloud दोनों के against काम करता है — सिर्फ़ URL बदलता है।

---

## Slide 4: Installation & Testing (Windows)

- MSI installer `clickhouse-odbc` के releases page से download करके run करना होता है।
- Test करने के लिए PowerShell script दिया गया है — जो `System.Data.Odbc.OdbcConnection` बना के `Url`, `Username`, `Password` pass करता है, connection open करता है, `select version()` query चलाता है, और result print करके connection close करता है।
- **Recommendation:** ClickHouse server को version **24.11 या उससे ऊपर** रखो — best driver compatibility के लिए।

---

## Slide 5: Configuration Parameters

| Parameter | क्या करता है |
|---|---|
| **Url** | ClickHouse server का पूरा HTTP(S) endpoint (protocol, host, port, path) |
| **Username / Password** | Authentication के लिए credentials |
| **Database** | Connection के लिए default database |
| **Timeout** | Driver कितनी देर तक server के response का wait करेगा (seconds में) |
| **ClientName** | Custom client identifier, User-Agent header में भेजता है |
| **Compression** | HTTP compression enable करता है — बड़े result sets के लिए bandwidth कम करने के लिए |
| **SqlCompatibilitySettings** | ClickHouse को traditional RDBMS जैसा behave करवाता है — Power BI जैसे tools के लिए |

---

## Slide 6: Connection String Examples

- **Local (WSL) ClickHouse:**
  `Driver={ClickHouse ODBC Driver (Unicode)};Url=http://localhost:8123/;Username=default`

- **ClickHouse Cloud:**
  `Driver={ClickHouse ODBC Driver (Unicode)};Url=https://your-instance.gcp.clickhouse.cloud:8443/;Username=default;Password=your-password`

दोनों examples में **same driver** use होता है — सिर्फ़ `Url` और credentials change होते हैं local vs cloud के बीच।

---

## Slide 7: Power BI Integration

Power BI के पास दो connectors हैं जो दोनों ODBC internally use करते हैं, लेकिन capabilities अलग हैं:

| | **ClickHouse Connector** (Recommended) | **Generic ODBC Connector** |
|---|---|---|
| Mode | DirectQuery support करता है | सिर्फ़ Import mode |
| कैसे काम करता है | Power BI खुद SQL generate करता है, सिर्फ़ ज़रूरी data लाता है हर visualization/filter के लिए | पूरा query (या पूरी table) execute करके पूरा result set import करता है |
| Best for | Interactive dashboards, बड़े datasets पे | जब पूरे data का local copy चाहिए हो |
| Refresh behavior | On-demand data fetch | हर refresh पे **पूरा dataset** दोबारा import होता है |

**मतलब:** ClickHouse Connector ज़्यादा efficient है बड़े data के लिए, क्योंकि वो सिर्फ़ जितना ज़रूरी है उतना ही fetch करता है।

---

## Slide 8: SQL Compatibility Settings

`SqlCompatibilitySettings` enable करने से third-party tools (जैसे Power BI) के generate किए हुए queries **standard SQL** जैसा behave करते हैं — ClickHouse के अपने defaults से हट के।

- **`cast_keep_nullable`** — Default में ClickHouse `Nullable` type को non-Nullable में convert नहीं करता, जिससे BI-generated `CAST(...)` queries nullable columns पे fail हो जाती हैं। ये setting `CAST` को nullability preserve करने देती है, standard SQL जैसे।
- **`prefer_column_name_to_alias`** — ClickHouse normally एक repeated name को उसी SELECT list के अंदर alias resolve कर देता है, जिससे एक aggregate accidentally nested aggregate बन सकता है। ये setting ClickHouse को underlying column prefer करने देती है — जैसा कि ज़्यादातर databases करते हैं।

⚠️ **Caveat:** अगर user `readonly = 1` पे है, तो वो कोई भी setting change नहीं कर सकता — चाहे SELECT query ही क्यों न हो। इसलिए `SqlCompatibilitySettings` उनके लिए error देगा (अगला slide इसको solve करता है)।

---

## Slide 9: Read-Only Users के लिए Fix

READONLY error से बचने के दो तरीके:

**Option 1 — Recommended:**
`readonly = 2` set करो — इससे settings change हो सकती हैं, लेकिन user data के लिए फिर भी read-only रहता है।
```sql
ALTER USER your_odbc_user MODIFY SETTING readonly = 2
```

**Option 2 — User पे driver की settings पहले से match कर दो:**
```sql
ALTER USER your_odbc_user MODIFY SETTING
    cast_keep_nullable = 1,
    prefer_column_name_to_alias = 1
```
इससे driver की कोशिश एक "no-op" बन जाती है क्योंकि settings पहले से ही वही हैं।

**Trade-off:** Option 2 में maintenance ज़्यादा है — future driver versions नए settings add कर सकते हैं जिन्हें manually mirror करना पड़ेगा।

---

## Summary (Slide 10)

- ODBC के through **standards-compliant** access मिलता है ClickHouse को — HTTP protocol के ऊपर, local और cloud दोनों में काम करता है।
- Configuration `Url`, `Username`, `Password`, `Database`, `Timeout`, `ClientName`, `Compression` से होती है।
- `SqlCompatibilitySettings` तब use करो जब BI tools (जैसे Power BI) non-ClickHouse-flavored SQL generate करते हैं।
- Power BI में possible हो तो **ClickHouse Connector (DirectQuery)** prefer करो, generic ODBC connector से बेहतर है।
- Read-only users के लिए `readonly = 2` set करो, ताकि compatibility-setting errors न आएं।
