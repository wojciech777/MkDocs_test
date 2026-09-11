# Signature verification

Every webhook delivery from Orion Platform 4.2 LTS carries an `X-Orion-Signature` header with an HMAC computed over the raw request body and the delivery timestamp. Verification is mandatory: the receiving address is public, it is not protected by any credential of yours, and the signature is the only proof that the payload really came from `orion-worker` and was not modified in transit.

An unverified endpoint accepts a forged `order.paid` event from anybody who learns the URL, which is enough to trigger a shipment or a settlement. Treat a delivery that fails verification the same way you treat an unauthenticated request: answer `401`, log the `X-Orion-Delivery` identifier, and drop the body.

## Header format

```text title="X-Orion-Signature header"
X-Orion-Signature: t=1778490764,v2=8f3c1a9d47b6e05c2f8a1d3b9e7c40f6a2d5b8c1e4f7a0d3b6c9e2f5a8d1b4c7
```

| Element | Description |
| --- | --- |
| `t` | Delivery timestamp, Unix seconds. It is the moment the signature was produced, not the event's `occurred_at`. |
| `v2` | `HMAC-SHA256` over the string `{timestamp}.{raw_body}`, hex encoded in lower case. This is the scheme you must use. |
| `v1` | `HMAC-SHA1` over the same string. Deprecated in 4.2 and removed in 5.0. |

The signing key is the endpoint's `whsec_...` secret, used as an ASCII byte string **including the `whsec_` prefix**. Do not strip the prefix, do not base64 decode the value, and do not trim it to the random part.

Elements are separated by commas and may appear in any order. A header may contain more than one `v2` element during a secret rotation, so parse the value into a list rather than assuming a single match.

!!! warning "v1 is deprecated"
    Integrations built against the 3.8 line verified `v1=` with `HMAC-SHA1`. That element is still present for backward compatibility in the whole 4.2 LTS line and disappears in 5.0. Switch to `v2` now; the two are computed over exactly the same signed string, so the migration touches only the digest algorithm. See [Migrate from 3.8 LTS to 4.2 LTS](../../guides/migrations/from-3-8-to-4-2.md).

## Signed string

The signed string is the timestamp, a single dot, and the request body exactly as it arrived on the wire:

```text title="Signed string"
1778490764.{"id":"evt_01HQ9AB7WZP4N6TKD2XR8M","type":"order.paid",...}
```

The dot is a literal separator and is not part of the body. Nothing else is signed: not the headers, not the path, not the query string.

## Verification algorithm

1. Read the raw body as bytes, before any JSON parsing or framework level transformation.
2. Split `X-Orion-Signature` on commas, then split each element on the first `=`. Collect `t` and every `v2` value.
3. Reject the delivery if `t` is missing, is not an integer, or differs from the current time by more than the tolerance of **300 s** in either direction.
4. Recompute `HMAC-SHA256` over the byte string `"{t}." + raw body bytes` for each secret you currently accept, and hex encode the result in lower case.
5. Compare each computed digest with each `v2` value from the header using a constant-time comparison.
6. Accept the delivery if any active secret produces a match. Otherwise answer `401` and do not process the body.

!!! danger "Raw bytes and constant-time comparison"
    Two mistakes account for almost every reported signature mismatch. First, verifying a re-serialized body: parsing JSON and dumping it again changes key order, whitespace, and number formatting, so the HMAC no longer matches. Capture the raw bytes and verify those. Second, comparing digests with `==` or `equals`: string comparison returns early on the first differing byte and leaks timing information that lets an attacker recover a valid signature byte by byte. Always use a constant-time primitive such as `MessageDigest.isEqual`, `hmac.compare_digest`, or `crypto.timingSafeEqual`.

## Implementations

Each implementation below accepts a collection of secrets, so that a rotation overlap period requires no code change.

=== "Java"

    ```java title="OrionSignatureVerifier.java" hl_lines="27 45 49"
    import javax.crypto.Mac;
    import javax.crypto.spec.SecretKeySpec;
    import java.nio.charset.StandardCharsets;
    import java.security.MessageDigest;
    import java.time.Instant;
    import java.util.ArrayList;
    import java.util.Collection;
    import java.util.List;

    public final class OrionSignatureVerifier {

        private static final long TOLERANCE_SECONDS = 300L;

        private final List<String> secrets;

        public OrionSignatureVerifier(Collection<String> secrets) {
            this.secrets = secrets.stream().filter(s -> s != null && !s.isBlank()).toList();
            if (this.secrets.isEmpty()) {
                throw new IllegalArgumentException("At least one whsec_ secret is required");
            }
        }

        public boolean isValid(String header, byte[] rawBody) {
            if (header == null) {
                return false;
            }
            String timestamp = null;
            List<String> signatures = new ArrayList<>();
            for (String element : header.split(",")) {
                String[] pair = element.trim().split("=", 2);
                if (pair.length != 2) {
                    continue;
                }
                if ("t".equals(pair[0])) {
                    timestamp = pair[1];
                } else if ("v2".equals(pair[0])) {
                    signatures.add(pair[1]);
                }
            }
            if (timestamp == null || signatures.isEmpty()) {
                return false;
            }

            long sent;
            try {
                sent = Long.parseLong(timestamp);
            } catch (NumberFormatException e) {
                return false;
            }
            if (Math.abs(Instant.now().getEpochSecond() - sent) > TOLERANCE_SECONDS) {
                return false;
            }

            for (String secret : secrets) {
                byte[] expected = hexToBytes(sign(secret, timestamp, rawBody));
                for (String signature : signatures) {
                    byte[] actual = hexToBytes(signature);
                    if (actual.length == expected.length && MessageDigest.isEqual(expected, actual)) {
                        return true;
                    }
                }
            }
            return false;
        }

        private static String sign(String secret, String timestamp, byte[] rawBody) {
            try {
                Mac mac = Mac.getInstance("HmacSHA256");
                mac.init(new SecretKeySpec(secret.getBytes(StandardCharsets.UTF_8), "HmacSHA256"));
                mac.update((timestamp + ".").getBytes(StandardCharsets.UTF_8));
                mac.update(rawBody);
                StringBuilder hex = new StringBuilder(64);
                for (byte b : mac.doFinal()) {
                    hex.append(String.format("%02x", b));
                }
                return hex.toString();
            } catch (Exception e) {
                throw new IllegalStateException("HMAC-SHA256 is not available", e);
            }
        }

        private static byte[] hexToBytes(String hex) {
            if (hex.length() % 2 != 0) {
                return new byte[0];
            }
            byte[] out = new byte[hex.length() / 2];
            for (int i = 0; i < out.length; i++) {
                out[i] = (byte) Integer.parseInt(hex.substring(i * 2, i * 2 + 2), 16);
            }
            return out;
        }
    }
    ```

    In a Spring MVC controller the body must be bound as `byte[]`, otherwise message converters rewrite it before you get to see it. `orion-sdk-java` ships the same algorithm as `WebhookVerifier`; see [Java SDK](../sdk/java.md).

=== "Python"

    ```python title="orion_signature.py" hl_lines="9 25 31"
    import hashlib
    import hmac
    import time
    from typing import Iterable

    TOLERANCE_SECONDS = 300


    def is_valid(header: str, raw_body: bytes, secrets: Iterable[str]) -> bool:
        if not header:
            return False

        timestamp = None
        signatures = []
        for element in header.split(","):
            name, _, value = element.strip().partition("=")
            if name == "t":
                timestamp = value
            elif name == "v2":
                signatures.append(value)

        if not timestamp or not signatures:
            return False
        if not timestamp.isdigit():
            return False
        if abs(int(time.time()) - int(timestamp)) > TOLERANCE_SECONDS:
            return False

        signed = timestamp.encode("ascii") + b"." + raw_body
        for secret in secrets:
            if not secret:
                continue
            expected = hmac.new(secret.encode("ascii"), signed, hashlib.sha256).hexdigest()
            for signature in signatures:
                if hmac.compare_digest(expected, signature):
                    return True
        return False
    ```

    With Flask, read the body through `request.get_data()`; with FastAPI, use `await request.body()`. Both return the untouched bytes. `request.json` and a Pydantic model do not.

=== "Node.js"

    ```javascript title="orionSignature.js" hl_lines="6 24 30"
    const crypto = require("node:crypto");

    const TOLERANCE_SECONDS = 300;

    function isValid(header, rawBody, secrets) {
      if (!header) {
        return false;
      }

      let timestamp = null;
      const signatures = [];
      for (const element of header.split(",")) {
        const index = element.indexOf("=");
        if (index < 0) {
          continue;
        }
        const name = element.slice(0, index).trim();
        const value = element.slice(index + 1).trim();
        if (name === "t") {
          timestamp = value;
        } else if (name === "v2") {
          signatures.push(value);
        }
      }

      if (!timestamp || !/^\d+$/.test(timestamp) || signatures.length === 0) {
        return false;
      }
      const now = Math.floor(Date.now() / 1000);
      if (Math.abs(now - Number(timestamp)) > TOLERANCE_SECONDS) {
        return false;
      }

      const signed = Buffer.concat([Buffer.from(`${timestamp}.`, "ascii"), rawBody]);
      for (const secret of secrets.filter(Boolean)) {
        const expected = crypto.createHmac("sha256", secret).update(signed).digest();
        for (const signature of signatures) {
          const actual = Buffer.from(signature, "hex");
          if (actual.length === expected.length && crypto.timingSafeEqual(expected, actual)) {
            return true;
          }
        }
      }
      return false;
    }

    module.exports = { isValid };
    ```

    In Express, mount `express.raw({ type: "application/json" })` on the webhook route only. A global `express.json()` middleware replaces `req.body` with a parsed object and the raw bytes are gone.

=== "curl"

    ```bash title="Recomputing a signature by hand"
    SECRET="whsec_9f2c7ab41de84c05b6e3a71d05"
    T=1778490764
    BODY_FILE=delivery.json

    printf '%s.' "$T" | cat - "$BODY_FILE" \
      | openssl dgst -sha256 -hmac "$SECRET" -hex
    ```

    ```text title="Expected output"
    HMAC-SHA256(stdin)= 8f3c1a9d47b6e05c2f8a1d3b9e7c40f6a2d5b8c1e4f7a0d3b6c9e2f5a8d1b4c7
    ```

    Save the delivery body to `delivery.json` byte for byte, without reformatting it, and compare the digest with the `v2` element of the header. This is the fastest way to tell a wrong secret apart from a modified body: if the manual digest matches the header but your application rejects the delivery, the body your code sees is not the body that was sent.

## Clock tolerance

The `t` value is checked against a tolerance of 300 s in both directions, which limits how long a captured delivery can be replayed. Two consequences follow:

- The receiving host must run NTP. A drift above five minutes rejects every delivery, including successful retries, and the endpoint gets suspended after the retry cycle is exhausted.
- Timestamp tolerance is not deduplication. A replay inside the window still needs the `evt_...` check described in [Event types](event-types.md).

## Secret rotation

Rotation issues a new secret while the previous one stays valid for an overlap period, so no delivery is lost and no deployment has to be timed precisely. The call needs the `admin` scope.

```bash title="Rotating the secret with a 24 h overlap"
curl -X POST \
  https://api.orion.example.com/v1/webhooks/whk_01HQ9B4TMZP7R3XK2VD8FE/rotate-secret \
  -H "X-Orion-Key: ok_live_9f2c41ab7d5e" \
  -H "Content-Type: application/json" \
  -d '{"overlap_seconds": 86400}'
```

```json title="200 OK response"
{
  "id": "whk_01HQ9B4TMZP7R3XK2VD8FE",
  "secret": "whsec_1b74de50a9c34f28b0e5c73a92",
  "previous_secret_expires_at": "2026-05-15T09:30:00Z",
  "rotated_at": "2026-05-14T09:30:00Z"
}
```

| Field | Default | Range | Description |
| --- | --- | --- | --- |
| `overlap_seconds` | `86400` | `0` to `604800` | How long the previous secret keeps signing deliveries. `0` invalidates it immediately. |

During the overlap period both secrets are active and a delivery is signed with each of them, which is why the header can contain two `v2` elements. A receiver that accepts a collection of secrets needs no change at all.

The recommended procedure:

1. Call `POST /v1/webhooks/{id}/rotate-secret` and store the returned `whsec_...` value in your secret store. It is returned only once, exactly like the value from the original `201 Created` response.
2. Add the new secret to the accepted list next to the old one and deploy. Verification must accept both values before you touch anything else.
3. Wait until the deliveries queued before the rotation have drained. A single event exhausts its eight attempts after roughly 2 min 10 s, as described in [Retries and idempotency](../../guides/events/retries.md), so a few minutes is enough for the in-flight traffic to be verified against a secret you still hold.
4. Remove the old secret after `previous_secret_expires_at` has passed. From that moment no delivery is signed with it any more.

!!! tip "Two environment variables, not one"
    Keep the accepted secrets in `ORION_WEBHOOK_SECRET` and `ORION_WEBHOOK_SECRET_PREVIOUS`, and let the verifier skip empty values. Rotation then becomes two configuration updates instead of a code change, which matches the overlap requirement in [Operational security](../../operations/security.md).

!!! danger "Never log the secret"
    A `whsec_...` value in an application log, an exception message, or an issue tracker attachment is a leaked credential and requires an immediate rotation with `overlap_seconds` set to `0`. Log the `X-Orion-Delivery` identifier and the computed digest prefix instead, never the key.

## Troubleshooting

Signature mismatch on every delivery, while the manual `openssl` check matches
:   A framework parsed and re-serialized the body before verification. Bind the body as `byte[]`, `bytes`, or a `Buffer` on the webhook route and verify those bytes. Watch out for global JSON middleware, gzip decompression by a proxy, and charset conversion.

Signature mismatch only for some deliveries
:   A rotation is in progress and your code accepts a single secret. Add the second value, or check whether the deliveries that fail carry two `v2` elements while your parser keeps only the first one.

Deliveries rejected with a timestamp outside the tolerance
:   Clock skew on the receiving host. Compare `t` with the local clock in the rejection log; a constant offset points at a missing or broken NTP client rather than at Orion.

Digest never matches, regardless of the payload
:   The secret is wrong. The most frequent cause is stripping the `whsec_` prefix, or a trailing newline picked up when the value was copied from a file or a console. The key is the full ASCII string, prefix included.

Verification passes but the header has no `v2`
:   The code is still reading `v1`. `HMAC-SHA1` is deprecated in 4.2 and gone in 5.0. Recompute with `HMAC-SHA256`; the signed string does not change.

Verification passes and the same order is processed twice
:   Signature verification is not deduplication. Store processed `evt_...` identifiers and compare `sequence`, as described in [Retries and idempotency](../../guides/events/retries.md).

## See also

- [Webhooks](index.md)
- [Event types](event-types.md)
- [Java SDK](../sdk/java.md)
- [Python SDK](../sdk/python.md)
- [Operational security](../../operations/security.md)
