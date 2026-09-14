# ClickHouse Keeper - सरल समझाइए 🎯

## **क्या है ClickHouse Keeper?**

Keeper एक tool है जो database को **replicate करने में मदद करता है** (यानी same data को कई servers पर रखना)। 

Think of it like ये - आपके पास एक किताब है और आप चाहते हो कि 3 libraries में same copy रहे। Keeper यही काम करता है!

---

## **इसके मुख्य काम (Core Jobs):**

1. **डेटा की जानकारी store करना** - कौन सा server कहाँ है, यह track करना
2. **सभी changes को note करना** - जब कोई नया data add हो तो सभी को बताना
3. **Leader choose करना** - कौन merge operation करेगा (expensive work को duplicate न करना)
4. **Check करना** - कौन सा server down है या healthy है

---

## **Replication क्यों जरूरी है?**

| फायदे | मतलब |
|-------|-------|
| **High Availability** | अगर एक server down हो तो दूसरे से data मिल जाएगा |
| **सभी को same data** | सभी servers पर एक जैसा डेटा रहे |
| **Smart Merging** | एक ही जगह काम करो, बाकी को copy दे दो |

---

## **Lab में 4 Tests हैं:**

### **Test 1: Basic Replication** ✓
```
INSERT data → Keeper log में जाता है → सभी servers को मिलता है
```

### **Test 2: Merge Coordination** ⚡
```
10 छोटे data pieces → 1 server merge करे → बाकी को copy मिले
(तीनों को अलग करने से अच्छा - सिर्फ एक को करो!)
```

### **Test 3: Leader Election** 🎓
```
अगर leader server down हो → नया leader choose हो → काम चलता रहे
```

### **Test 4: Automatic Catch-Up** 🔄
```
Server down था → फिर से on किया → Keeper से सभी missed changes replay हो जाते हैं
```

---

## **महत्वपूर्ण बातें:**

✅ **Keeper ही सच्चाई है** - सभी servers को trust करना चाहिए  
✅ **Merges efficiently होते हैं** - एक server काम करे, बाकी lazy हों  
✅ **Automatic recovery** - server down हो गया? कोई बात नहीं, restart पर automatic ठीक हो जाएगा  
✅ **कोई split-brain नहीं** - सभी कुछ conflict नहीं करते  

---

## **Setup:**
```
1 Keeper + 3 ClickHouse Servers = पूरा distributed system
```

यह lab में आप सभी tests run करके देख सकते हो! 🚀
