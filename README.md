 
**saker-kommunikation**

---

# **TLS, autentisering och säker kommunikation**

## **Översikt**  
Idag fokuserar vi på att säkra en kommunikationsväg i IoT‑caset genom kryptering, certifikatverifiering och åtkomstkontroll. Arbetet demonstrerar hur en klient verifierar en broker eller tjänst, hur felaktiga identiteter stoppas och hur behörighet skiljs från transportskydd. Två separata demos används: MQTTS och HTTPS.

---

## **MQTTS‑demo (Mosquitto över TLS)**  
Demot kör en lokal Mosquitto‑broker med TLS på port 8883. En lokal CA signerar brokerns servercertifikat. Publisher och subscriber verifierar brokern med CA‑certifikatet och genomför två tester.

### **Körning**
```
cd demo_mqtts
bash run_demo.sh
```

### **Förväntat resultat**
1. Klienten litar på fel CA → TLS‑handshake stoppas  
2. Klienten litar på rätt CA → JSON‑telemetri levereras  
3. Summering: 2/2 expected outcomes

### **Certifikatsinspektion**
```
openssl x509 -in generated/server.crt -noout -subject -issuer -dates -ext subjectAltName
```

**Observerat:**  
- subject: CN=localhost  
- issuer: IOT25 MQTT Demo CA  
- giltighetstid: två dagar (demo)  
- SAN: localhost och 127.0.0.1  

**Slutsats:**  
Klienten får endast använda certifikatet för localhost/127.0.0.1 och måste ha rätt CA för att verifiera brokern.

---

## **HTTPS‑demo (TLS + API‑nyckelkontroll)**  
Demot visar skillnaden mellan transportskydd (TLS) och behörighet (API‑nyckel).

### **Körning**
```
cd demo_tls
bash run_demo.sh
```

### **Förväntat resultat**
1. Fel CA → TLS_VERIFY_FAILED  
2. Rätt CA men saknad API‑nyckel → HTTP 401  
3. Rätt CA och korrekt API‑nyckel → HTTP 200 med JSON  
4. Summering: 3/3 expected outcomes

**Slutsats:**  
TLS verifierar serverns identitet. API‑nyckeln styr åtkomst. Dessa är separata säkerhetslager.

---

## **Hotmodell**

| Hot | Konsekvens | Kontroll | Test | Begränsning |
|-----|------------|----------|------|-------------|
| Avlyssning | Telemetri läcks | TLS | Fel CA → TLS_VERIFY_FAILED | Certifikatrotation krävs |
| Falsk server | MITM‑risk | Certifikatkontroll | Fel CA → stopp före MQTT | Klienten måste ha rätt CA |
| Obehörig klient | Fel topic eller API‑anrop | API‑nyckel | HTTP 401 | Nycklar kan läcka |
| Läckta autentiseringsuppgifter | Angripare får åtkomst | Nyckel utanför Git | Fel API‑nyckel → 401 | Kräver säker lagring |
| Utgånget certifikat | Klienter kan inte ansluta | Rotation | Certifikatets notAfter | OTA måste fungera |

---

## **Testprotokoll**

| Test | Förutsättning | Förväntat | Observerat | Godkänt |
|------|--------------|-----------|------------|---------|
| MQTTS med fel CA | Broker körs | TLS stoppas | TLS_VERIFY_FAILED | Ja |
| MQTTS med rätt CA | Publisher/subscriber körs | JSON levereras | JSON mottaget | Ja |
| Betrodd TLS‑server och rätt uppgift | Rätt CA + rätt API‑nyckel | Data returneras | HTTP 200 | Ja |
| Servercertifikat utan betrodd CA | Fel CA | TLS stoppas | TLS_VERIFY_FAILED | Ja |
| Betrodd TLS‑server utan uppgift | Rätt CA | HTTP 401 | HTTP 401 | Ja |
| Betrodd TLS‑server med fel uppgift | Rätt CA + fel API‑nyckel | HTTP 401 | HTTP 401 | Ja |

---

## **Hemlighetshantering**

**Lagringsplatser:**  
- CA‑certifikat: firmware (offentligt)  
- API‑nyckel: miljövariabel eller separat fil utanför Git  
- Privat servernyckel: endast på broker/API‑server  

**Undantag från Git:**  
- `.gitignore` för `generated/`  
- API‑nyckel i `.env` eller separat fil  

**Införande av ny uppgift:**  
- Uppdatera API‑nyckel i servern  
- Distribuera ny CA via OTA  

**Återkallelse vid läckage:**  
- Rotera API‑nyckeln  
- Byt CA och servercertifikat  
- Ta bort gammal CA efter överlappning

---

## **Slutsats**  
Detta visar hur en IoT‑kommunikationsväg skyddas med TLS, certifikatverifiering och åtkomstkontroll. Tester demonstrerar både godkända och nekade flöden. Dokumentationen skiljer tydligt mellan kryptering, autentisering och auktorisation. Hemligheter hanteras utanför Git och certifikatrotation planeras för långsiktig drift.
