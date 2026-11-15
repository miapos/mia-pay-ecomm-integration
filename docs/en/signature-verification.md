# Signature Verification Guide

---

## Description

This document describes the process of verifying the signature of data received from the eComm service in JSON format.
The RSA algorithm is used for signing, and the signature is created with a private key. Merchants can verify the
authenticity of the data using the public key provided separately.

---

## Steps for Signature Verification

### 1. Receiving the JSON with Payment Results

After processing a payment, the eComm service sends a JSON with the payment results to the `callbackUrl` specified by
the merchant during payment initiation.

**Example JSON:**

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

### 2. Obtaining the Public Key

The public key can be obtained by making the following request to the eComm service:

- **Method:** `GET`
- **Endpoint:** `/ecomm/api/v1/public-key`
- **Header:** `Authorization: Bearer <accessToken>`, the token from `POST /ecomm/api/v1/token`. The test environment and every bank have their own key.

**Example Response:**

```json
{
  "publicKey": "MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAr1f/yhw+UI//z3KdpnJz..."
}
```

---

### 3. Constructing the String for Signature Verification

To create the string used for verifying the signature, follow these steps:

1. Extract the `result` object from the JSON.
2. Sort all fields of the `result` object alphabetically.
3. Convert each value to a string. `amount` is always written with exactly two decimals (`1775` → `1775.00`, `12.5` → `12.50`); every other value is used as received.
4. Concatenate the values of all fields using `;` as the separator.

**Example Result:**

```
145.25;MDL;order123;2024-05-20T16:32:28;bc340d13-7411-4785-a083-b594b1384eb5;qr;SUCCESS;swift123;SomeBank;123456
```

---

### 4. Verifying the Signature

Use the public key and RSA algorithm to verify the signature. Compare the computed signature with the one provided in
the `signature` field.

---

## Code Examples

### Example in Java

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

### Example in Python

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

## Conclusion

By following the steps and code examples provided above, you can verify the authenticity of the data received from the
eComm service and ensure that it was signed using the private key of the eComm service. This guarantees that the data
has not been modified or tampered with.
