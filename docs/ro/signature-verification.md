# Ghid de verificare a semnăturii

---

## Descriere

Acest document descrie procesul de verificare a semnăturii datelor primite de la serviciul eComm în format JSON. Pentru
semnare este utilizat algoritmul RSA, iar semnătura este creată folosind o cheie privată. Comercianții pot verifica
autenticitatea datelor utilizând cheia publică, care este furnizată separat.

---

## Pașii pentru verificarea semnăturii

### 1. Primirea JSON-ului cu rezultatele plății

După procesarea plății, serviciul eComm trimite un JSON cu rezultatele plății către `callbackUrl`, specificat de
comerciant la inițierea plății.

**Exemplu JSON:**

```json
{
  "result": {
    "terminalId": "123456",
    "orderId": "order123",
    "paymentId": "bc340d13-7411-4785-a083-b594b1384eb5",
    "status": "SUCCESS",
    "amount": 145.25,
    "currency": "MDL",
    "paymentType": "qr",
    "paymentDate": "2024-05-20T16:32:28",
    "swiftMessageId": "swift123",
    "swiftPayerBank": "SomeBank"
  },
  "signature": "base64_encoded_signature"
}
```

---

### 2. Obținerea cheii publice

Cheia publică poate fi obținută prin următoarea solicitare către serviciul eComm:

- **Metodă:** `GET`
- **Endpoint:** `/ecomm/api/v1/public-key`
- **Antet:** `Authorization: Bearer <accessToken>`, tokenul obținut prin `POST /ecomm/api/v1/token`. Mediul de test și fiecare bancă au cheia lor.

**Exemplu de răspuns:**

```json
{
  "publicKey": "MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAr1f/yhw+UI//z3KdpnJz..."
}
```

---

### 3. Crearea unui șir pentru verificarea semnăturii

Pentru a crea un șir utilizat pentru verificarea semnăturii, urmați pașii de mai jos:

1. Extrageți obiectul `result` din JSON.
2. Sortează toate câmpurile obiectului `result` în ordine alfabetică.
3. Transformă fiecare valoare în șir. `amount` se scrie întotdeauna cu exact două zecimale (`1775` → `1775.00`, `12.5` → `12.50`); celelalte valori se folosesc așa cum au venit.
4. Concatenează valorile tuturor câmpurilor folosind `;` ca separator.

**Exemplu rezultat:**

```
145.25;MDL;order123;2024-05-20T16:32:28;bc340d13-7411-4785-a083-b594b1384eb5;qr;SUCCESS;swift123;SomeBank;123456
```

---

### 4. Verificarea semnăturii

Pentru verificarea semnăturii, utilizați cheia publică și algoritmul RSA. Comparați semnătura calculată cu cea oferită
în câmpul `signature`.

---

## Exemple de cod

### Exemplu în Java

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.nio.charset.StandardCharsets;
import java.security.KeyFactory;
import java.security.PublicKey;
import java.security.Signature;
import java.security.spec.X509EncodedKeySpec;
import java.util.Base64;
import java.util.Map;
import java.util.TreeMap;
import java.util.stream.Collectors;

public class SignatureVerification {
    public static boolean verifySignature(String jsonString, String publicKeyPEM) {
        try {
            ObjectMapper objectMapper = new ObjectMapper();
            Map<String, Object> jsonMap = objectMapper.readValue(jsonString, Map.class);
            Map<String, Object> resultMap = (Map<String, Object>) jsonMap.get("result");
            String base64Signature = (String) jsonMap.get("signature");

            Map<String, Object> sortedMap = new TreeMap<>(resultMap);
            // amount is always signed with two decimals: 1775 -> 1775.00
            String signatureString = sortedMap.entrySet().stream()
                    .map(e -> "amount".equals(e.getKey())
                            ? new BigDecimal(e.getValue().toString()).setScale(2, RoundingMode.HALF_UP).toPlainString()
                            : e.getValue().toString())
                    .collect(Collectors.joining(";"));

            byte[] decodedKey = Base64.getDecoder().decode(publicKeyPEM);
            X509EncodedKeySpec keySpec = new X509EncodedKeySpec(decodedKey);
            KeyFactory keyFactory = KeyFactory.getInstance("RSA");
            PublicKey publicKey = keyFactory.generatePublic(keySpec);

            Signature signature = Signature.getInstance("SHA256withRSA");
            signature.initVerify(publicKey);
            signature.update(signatureString.getBytes(StandardCharsets.UTF_8));

            byte[] signatureBytes = Base64.getDecoder().decode(base64Signature);
            return signature.verify(signatureBytes);
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }
}
```

### Exemplu în Python

```python
import json
import base64
from decimal import Decimal
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import padding
from cryptography.hazmat.primitives import serialization

def verify_signature(json_string, public_key_base64):
    try:
        json_data = json.loads(json_string, parse_float=Decimal)
        result = json_data['result']
        signature_base64 = json_data['signature']

        sorted_items = sorted(result.items())
        # amount is always signed with two decimals: 1775 -> 1775.00
        signature_string = ';'.join(
            f"{Decimal(str(value)):.2f}" if key == 'amount' else str(value)
            for key, value in sorted_items
        )

        public_key = serialization.load_pem_public_key(
            f"-----BEGIN PUBLIC KEY-----\n{public_key_base64}\n-----END PUBLIC KEY-----".encode()
        )

        signature = base64.b64decode(signature_base64)
        public_key.verify(
            signature,
            signature_string.encode("utf-8"),
            padding.PKCS1v15(),
            hashes.SHA256()
        )
        return True
    except Exception as e:
        print(f"Signature verification failed: {e}")
        return False
```

---

## Concluzie

Folosind pașii și exemplele de cod descrise mai sus, puteți verifica autenticitatea datelor primite de la serviciul
eComm și vă puteți asigura că acestea au fost semnate utilizând cheia privată a serviciului eComm. Aceasta garantează că
datele nu au fost modificate sau falsificate.
