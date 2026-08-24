# Formidable eSign

Sign a document without sending the document or its URL to Formidable eSign.

## 1. Hash the document locally

```bash
DOCUMENT_HASH=$(openssl dgst -sha256 -r document.pdf | cut -d' ' -f1)
echo "$DOCUMENT_HASH"
```

The result is a 64-character SHA-256 hash. The same document bytes always
produce the same hash; any change produces a different hash.

## 2. Sign the hash

```bash
SIGNATURE=$(
  jq -n \
    --arg hash "$DOCUMENT_HASH" \
    --arg certId "$FESIGN_CERT_ID" \
    --arg pin "$FESIGN_PIN" \
    '{hash: $hash, certId: $certId, pin: $pin}' |
  curl --silent --fail-with-body \
    --request POST "$FESIGN_API_URL/documents/signHash" \
    --header "x-api-key: $FESIGN_API_KEY" \
    --header "Content-Type: application/json" \
    --data-binary @- |
  jq -r '.signature'
)
```

Only the hash is signed. Do not include the document URL, filename, or document
contents in the request or optional metadata.

## 3. Verify with Formidable eSign

```bash
jq -n \
  --arg hash "$DOCUMENT_HASH" \
  --arg signature "$SIGNATURE" \
  '{hash: $hash, signature: $signature}' |
curl --silent --fail-with-body \
  --request POST "$FESIGN_API_URL/documents/validate" \
  --header "x-api-key: $FESIGN_API_KEY" \
  --header "Content-Type: application/json" \
  --data-binary @-
```

A valid response returns `"isValid": true` and the certificate, chain, hash,
and signature checks.

## 4. Verify locally

Decode the returned signature, fetch the Formidable eSign trust anchor, and
verify the original document with OpenSSL:

```bash
printf '%s' "$SIGNATURE" |
  base64 --decode |
  jq -r '.signature' |
  base64 --decode > signature.p7s

curl --silent --fail-with-body \
  "$FESIGN_API_URL/certificates/root-ca" \
  --header "x-api-key: $FESIGN_API_KEY" \
  --output fesign-root-ca.crt

openssl cms -verify \
  -binary \
  -inform DER \
  -in signature.p7s \
  -content document.pdf \
  -CAfile fesign-root-ca.crt \
  -purpose any \
  -out /dev/null
```

OpenSSL exits successfully only when the document hash, signature, and
certificate chain are valid.

## More

- [Formidable eSign technical whitepaper](https://formidable.care/esign/whitepaper)
- [Request Formidable eSign API access](https://formidable.care/contact?interest=esign)
