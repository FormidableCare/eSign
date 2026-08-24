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

## JavaScript (Node.js)

Uses Node.js 18+ built-ins: [`node:crypto`](https://nodejs.org/api/crypto.html),
`fetch`, and `node:child_process`. OpenSSL is used for local CMS verification.
No npm package is required.

```javascript
// esign.mjs
import { createHash } from "node:crypto";
import { readFile, writeFile } from "node:fs/promises";
import { spawnSync } from "node:child_process";

const apiUrl = required("FESIGN_API_URL").replace(/\/$/, "");
const apiKey = required("FESIGN_API_KEY");
const certId = required("FESIGN_CERT_ID");
const pin = required("FESIGN_PIN");
const documentPath = "document.pdf";

// 1. Hash locally.
const document = await readFile(documentPath);
const hash = createHash("sha256").update(document).digest("hex");
console.log({ hash });

// 2. Sign only the hash. No document or URL is sent.
const signed = await post("/documents/signHash", { hash, certId, pin });
const signature = signed.signature;

// 3. Verify with Formidable eSign.
const verification = await post("/documents/validate", { hash, signature });
if (!verification.isValid) throw new Error("Formidable eSign verification failed");
console.log("Formidable eSign verification: valid");

// 4. Verify locally with OpenSSL.
const envelope = JSON.parse(Buffer.from(signature, "base64").toString("utf8"));
await writeFile("signature.p7s", Buffer.from(envelope.signature, "base64"));

const rootResponse = await fetch(`${apiUrl}/certificates/root-ca`, {
  headers: { "x-api-key": apiKey },
});
if (!rootResponse.ok) throw new Error(await rootResponse.text());
await writeFile("fesign-root-ca.crt", Buffer.from(await rootResponse.arrayBuffer()));

const nullDevice = process.platform === "win32" ? "NUL" : "/dev/null";
const openssl = spawnSync("openssl", [
  "cms", "-verify", "-binary", "-inform", "DER",
  "-in", "signature.p7s", "-content", documentPath,
  "-CAfile", "fesign-root-ca.crt", "-purpose", "any",
  "-out", nullDevice,
], { stdio: "inherit" });

if (openssl.status !== 0) throw new Error("Local verification failed");
console.log("Local verification: valid");

async function post(path, body) {
  const response = await fetch(`${apiUrl}${path}`, {
    method: "POST",
    headers: {
      "x-api-key": apiKey,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(body),
  });
  if (!response.ok) throw new Error(await response.text());
  return response.json();
}

function required(name) {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}
```

```bash
node esign.mjs
```

## .NET (C#)

Uses `SHA256`, `HttpClient`, and
[`SignedCms`](https://learn.microsoft.com/dotnet/api/system.security.cryptography.pkcs.signedcms).
Add the PKCS package:

```bash
dotnet add package System.Security.Cryptography.Pkcs
```

```csharp
// Program.cs — .NET 8+
using System.Net.Http.Json;
using System.Security.Cryptography;
using System.Security.Cryptography.Pkcs;
using System.Security.Cryptography.X509Certificates;
using System.Text.Json;

var apiUrl = Required("FESIGN_API_URL").TrimEnd('/') + "/";
var apiKey = Required("FESIGN_API_KEY");
var certId = Required("FESIGN_CERT_ID");
var pin = Required("FESIGN_PIN");
const string documentPath = "document.pdf";

using var client = new HttpClient { BaseAddress = new Uri(apiUrl) };
client.DefaultRequestHeaders.Add("x-api-key", apiKey);

// 1. Hash locally.
var document = await File.ReadAllBytesAsync(documentPath);
var hash = Convert.ToHexString(SHA256.HashData(document)).ToLowerInvariant();
Console.WriteLine($"Hash: {hash}");

// 2. Sign only the hash. No document or URL is sent.
using var signResponse = await client.PostAsJsonAsync(
    "documents/signHash",
    new { hash, certId, pin });
signResponse.EnsureSuccessStatusCode();

using var signJson = JsonDocument.Parse(await signResponse.Content.ReadAsStreamAsync());
var signature = signJson.RootElement.GetProperty("signature").GetString()
    ?? throw new InvalidOperationException("Signature missing");

// 3. Verify with Formidable eSign.
using var verifyResponse = await client.PostAsJsonAsync(
    "documents/validate",
    new { hash, signature });
verifyResponse.EnsureSuccessStatusCode();

using var verifyJson = JsonDocument.Parse(await verifyResponse.Content.ReadAsStreamAsync());
if (!verifyJson.RootElement.GetProperty("isValid").GetBoolean())
    throw new CryptographicException("Formidable eSign verification failed");
Console.WriteLine("Formidable eSign verification: valid");

// 4. Verify the detached CMS signature and certificate chain locally.
using var envelope = JsonDocument.Parse(Convert.FromBase64String(signature));
var cmsBytes = Convert.FromBase64String(
    envelope.RootElement.GetProperty("signature").GetString()
    ?? throw new InvalidOperationException("CMS signature missing"));

var cms = new SignedCms(new ContentInfo(document), detached: true);
cms.Decode(cmsBytes);
cms.CheckSignature(verifySignatureOnly: true);

var rootPem = await client.GetStringAsync("certificates/root-ca");
using var root = X509Certificate2.CreateFromPem(rootPem);
var signer = cms.SignerInfos[0].Certificate
    ?? throw new CryptographicException("Signer certificate missing");

using var chain = new X509Chain();
chain.ChainPolicy.TrustMode = X509ChainTrustMode.CustomRootTrust;
chain.ChainPolicy.CustomTrustStore.Add(root);
chain.ChainPolicy.ExtraStore.AddRange(cms.Certificates);
chain.ChainPolicy.RevocationMode = X509RevocationMode.NoCheck;

if (!chain.Build(signer))
    throw new CryptographicException("Certificate chain verification failed");
Console.WriteLine("Local verification: valid");

static string Required(string name) =>
    Environment.GetEnvironmentVariable(name)
    ?? throw new InvalidOperationException($"Missing {name}");
```

```bash
dotnet run
```

## More

- [Formidable eSign technical whitepaper](https://formidable.care/esign/whitepaper)
- [Request Formidable eSign API access](https://formidable.care/contact?interest=esign)
