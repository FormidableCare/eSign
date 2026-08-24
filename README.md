# Formidable eSign

Formidable eSign supports two document-signing flows:

| Document | Result | What is sent to Formidable eSign |
|---|---|---|
| JSON / FHIR | Detached CMS signature | SHA-256 hash only |
| PDF | PDF with embedded CMS signature and CA chain | PDF bytes, never a document URL |

Set these values before running the examples:

```bash
export FESIGN_API_URL="https://api.fesign.formidable.care"
export FESIGN_API_KEY="..."
export FESIGN_CERT_ID="..."
export FESIGN_PIN="..."
```

## Sign a JSON / FHIR document

Hash the exact UTF-8 file bytes locally. JSON whitespace and property order are
part of those bytes, so preserve the signed file or agree on a canonical JSON
format before hashing reconstructed objects.

### Hash and sign

```bash
DOCUMENT_HASH=$(openssl dgst -sha256 -r patient.fhir.json | cut -d' ' -f1)

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

Only the hash is sent. Do not put the document, filename, or URL in metadata.

### Verify with Formidable eSign

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

A valid response returns `"isValid": true`.

### Verify locally

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
  -content patient.fhir.json \
  -CAfile fesign-root-ca.crt \
  -purpose any \
  -out /dev/null
```

## Sign a PDF

`signPDF` receives the PDF, signs its PDF `ByteRange`, and returns a PDF with the
signature and certificate chain embedded. No document URL is used.

```bash
curl --silent --fail-with-body \
  --request POST "$FESIGN_API_URL/documents/signPDF" \
  --header "x-api-key: $FESIGN_API_KEY" \
  --form "pdf=@document.pdf;type=application/pdf" \
  --form "certId=$FESIGN_CERT_ID" \
  --form "pin=$FESIGN_PIN" |
jq -r '.signedPdf' |
base64 --decode > signed-document.pdf
```

The PDF limit is 10 MB. Verify the result in Adobe Acrobat or with
[`pdfsig`](https://manpages.debian.org/pdfsig) after trusting the Formidable
eSign Root CA:

```bash
pdfsig signed-document.pdf
```

## JavaScript (Node.js)

Uses Node.js 20+ built-ins: [`node:crypto`](https://nodejs.org/api/crypto.html),
`fetch`, `FormData`, and `Blob`. No npm package is required.

```javascript
// esign.mjs
import { createHash } from "node:crypto";
import { readFile, writeFile } from "node:fs/promises";

const apiUrl = required("FESIGN_API_URL").replace(/\/$/, "");
const apiKey = required("FESIGN_API_KEY");
const certId = required("FESIGN_CERT_ID");
const pin = required("FESIGN_PIN");

// JSON / FHIR: hash locally, sign only the hash, then verify it.
const fhir = await readFile("patient.fhir.json");
const hash = createHash("sha256").update(fhir).digest("hex");

const signedHash = await postJson("/documents/signHash", {
  hash,
  certId,
  pin,
});

const verification = await postJson("/documents/validate", {
  hash,
  signature: signedHash.signature,
});

if (!verification.isValid) throw new Error("FHIR signature is invalid");
await writeFile("patient.fhir.signature", signedHash.signature);

// PDF: upload the bytes and save the returned PDF with its embedded signature.
const pdf = await readFile("document.pdf");
const form = new FormData();
form.append("pdf", new Blob([pdf], { type: "application/pdf" }), "document.pdf");
form.append("certId", certId);
form.append("pin", pin);

const pdfResponse = await fetch(`${apiUrl}/documents/signPDF`, {
  method: "POST",
  headers: { "x-api-key": apiKey },
  body: form,
});
if (!pdfResponse.ok) throw new Error(await pdfResponse.text());

const signedPdf = await pdfResponse.json();
await writeFile("signed-document.pdf", Buffer.from(signedPdf.signedPdf, "base64"));

async function postJson(path, body) {
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
pdfsig signed-document.pdf
```

## .NET (C#)

Uses `SHA256`, `HttpClient`, and
[`SignedCms`](https://learn.microsoft.com/dotnet/api/system.security.cryptography.pkcs.signedcms).
Add the PKCS package for local JSON/FHIR verification:

```bash
dotnet add package System.Security.Cryptography.Pkcs
```

```csharp
// Program.cs — .NET 8+
using System.Net.Http.Headers;
using System.Net.Http.Json;
using System.Security.Cryptography;
using System.Security.Cryptography.Pkcs;
using System.Security.Cryptography.X509Certificates;
using System.Text.Json;

var apiUrl = Required("FESIGN_API_URL").TrimEnd('/') + "/";
var apiKey = Required("FESIGN_API_KEY");
var certId = Required("FESIGN_CERT_ID");
var pin = Required("FESIGN_PIN");

using var client = new HttpClient { BaseAddress = new Uri(apiUrl) };
client.DefaultRequestHeaders.Add("x-api-key", apiKey);

// JSON / FHIR: hash locally and sign only the hash.
var fhir = await File.ReadAllBytesAsync("patient.fhir.json");
var hash = Convert.ToHexString(SHA256.HashData(fhir)).ToLowerInvariant();

using var signResponse = await client.PostAsJsonAsync(
    "documents/signHash",
    new { hash, certId, pin });
signResponse.EnsureSuccessStatusCode();

using var signJson = JsonDocument.Parse(await signResponse.Content.ReadAsStreamAsync());
var signature = signJson.RootElement.GetProperty("signature").GetString()
    ?? throw new CryptographicException("Signature missing");
await File.WriteAllTextAsync("patient.fhir.signature", signature);

// Verify with Formidable eSign.
using var verifyResponse = await client.PostAsJsonAsync(
    "documents/validate",
    new { hash, signature });
verifyResponse.EnsureSuccessStatusCode();

using var verifyJson = JsonDocument.Parse(await verifyResponse.Content.ReadAsStreamAsync());
if (!verifyJson.RootElement.GetProperty("isValid").GetBoolean())
    throw new CryptographicException("FHIR signature is invalid");

// Verify the detached CMS signature and its certificate chain locally.
using var envelope = JsonDocument.Parse(Convert.FromBase64String(signature));
var cmsBytes = Convert.FromBase64String(
    envelope.RootElement.GetProperty("signature").GetString()
    ?? throw new CryptographicException("CMS signature missing"));

var cms = new SignedCms(new ContentInfo(fhir), detached: true);
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
    throw new CryptographicException("Certificate chain is invalid");

// PDF: upload the bytes and save the PDF with its embedded signature.
var pdf = await File.ReadAllBytesAsync("document.pdf");
using var pdfContent = new ByteArrayContent(pdf);
pdfContent.Headers.ContentType = new MediaTypeHeaderValue("application/pdf");

using var form = new MultipartFormDataContent
{
    { pdfContent, "pdf", "document.pdf" },
    { new StringContent(certId), "certId" },
    { new StringContent(pin), "pin" },
};

using var pdfResponse = await client.PostAsync("documents/signPDF", form);
pdfResponse.EnsureSuccessStatusCode();

using var pdfJson = JsonDocument.Parse(await pdfResponse.Content.ReadAsStreamAsync());
var signedPdf = Convert.FromBase64String(
    pdfJson.RootElement.GetProperty("signedPdf").GetString()
    ?? throw new CryptographicException("Signed PDF missing"));
await File.WriteAllBytesAsync("signed-document.pdf", signedPdf);

static string Required(string name) =>
    Environment.GetEnvironmentVariable(name)
    ?? throw new InvalidOperationException($"Missing {name}");
```

```bash
dotnet run
pdfsig signed-document.pdf
```

## More

- [Formidable eSign technical whitepaper](https://formidable.care/esign/whitepaper)
- [Request Formidable eSign API access](https://formidable.care/contact?interest=esign)
