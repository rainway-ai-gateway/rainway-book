# Chapter 23: Domain and Certificate Configuration

## Chapter Goals

Through this chapter, readers will learn:

- Understand the status and limitations of domain management in the current release;
- Upload TLS server certificates and private keys, and understand the Control Plane validation rules;
- Set up and maintain the Default Certificate so that HTTPS handshakes never fail;
- Track certificate expiration and plan renewal and replacement workflows;
- Verify that HTTPS access works and troubleshoot common certificate mismatch issues.

---

## Status of Domain Management

> **Important Note**: The current release of Rainway AI Gateway does not expose domain management functionality. Although the interface implementations under `ai-gateway-api/endpoints/openapi_v1/domain/` still exist, their Endpoint registration has been removed (`Endpoints = []*xreq.Endpoint{}` in `endpoints.go`), and `create.go` explicitly states `deprecated, endpoint registration removed per optimization plan v1.2`. Therefore, neither Dashboard nor OpenAPI currently provides the `/domains` interface.

In AI gateway scenarios, domain binding itself does not participate in AI route selection. The request forwarding target is determined entirely by the three-tier AI routing rules (API-Key / Entity / Global) in `mod_ai_route`; when BFE's `findProduct()` finds no matching product by hostname, it falls back to the default product line (`defaultProduct`, corresponding to `AIRouteInnerProductName` on the Control Plane), thereby loading the AI module configuration under the default product line. Since the current release works normally without maintaining custom domains, domain management is not yet open to users.

If domain management is reopened in a future release, its design will follow the traditional BFE HostTable mechanism:

```
hostname (e.g. api.example.com)
    │
    ▼
host-tag (e.g. host-tag-1)
    │
    ▼
product (e.g. AI_product)
```

- **Hosts**: mapping from `host-tag` to a list of `hostname`s;
- **HostTags**: mapping from `product` to a list of `host-tag`s;
- **DefaultProduct**: the fallback product line when no domain matches.

Note that at that point: AI module configuration is only associated with the default AI product line; if a custom domain is bound to a non-default product line, the AI module rules will not take effect.

---

---

## TLS Certificate Upload

HTTPS access depends on a server certificate and a private key. The current Dashboard also does not provide a certificate management UI; certificates must be maintained directly via the OpenAPI `/certificates` endpoints. The Control Plane is responsible for validation, storage, version control, and pushing the certificate path mapping to the Data Plane via InnerAPI.

### Certificate Data Model

Creating a certificate requires submitting the following fields:

| Field | Type | Description |
|------|------|------|
| `cert_name` | string | Certificate name, globally unique, 2-64 characters, letters, digits, `_`, `-`, `.` only |
| `description` | string | Certificate description, 2-256 characters |
| `is_default` | bool | Whether this is the default certificate; there must be exactly one default certificate globally |
| `cert_file_content` | string | Certificate file content, PEM-format X.509 certificate |
| `key_file_content` | string | Private key file content, PEM-format private key |
| `expired_date` | string | Expiration time, read-only, parsed by the server from the certificate content |

```json
{
  "cert_name": "cert_demo",
  "description": "api.example.com production certificate",
  "is_default": true,
  "cert_file_content": "-----BEGIN CERTIFICATE-----\nMIIDXTCCAkWgAwIBAgIJAKoK/heBjcOuMA0GCSqGSIb3DQEBCwUAMEUxCzAJBgNV\n...\n-----END CERTIFICATE-----",
  "key_file_content": "-----BEGIN RSA PRIVATE KEY-----\nMIIEpQIBAAKCAQEA3Tz2mvC3D1tRkX7B8kYwQ8u8kYwQ8u8...\n-----END RSA PRIVATE KEY-----"
}
```

### Create Certificate Endpoint

- **Endpoint**: `POST /open-api/v1/certificates`
- **Required fields**: `cert_name`, `description`, `is_default`, `cert_file_content`, `key_file_content`
- **Validation rules**:
  - The certificate and private key must be in valid PEM format;
  - The certificate and private key must be a matching pair; the server verifies that they match;
  - If the system has no default certificate yet, the new certificate must be set as the default;
  - `cert_name` must not be duplicated, and must not be named `BFE_DEFAULT_CERT`.

On success, the endpoint returns the certificate metadata but **does not return** the certificate or private key content:

```json
{
  "ErrNum": 200,
  "ErrMsg": "success",
  "Data": {
    "cert_name": "cert_demo",
    "description": "api.example.com production certificate",
    "is_default": true,
    "expired_date": "2026-08-23 16:02:31"
  }
}
```

If validation fails, the endpoint returns a clear error message. Common failure causes include: certificate and private key mismatch, invalid PEM format, `cert_name` not conforming to the naming rules or already existing, no default certificate set while the system has none, etc. In automation scripts, it is recommended to catch these errors and log the original `ErrMsg` returned, so problems can be located quickly.

### Certificate List and Details

- `GET /open-api/v1/certificates`: returns the full certificate list, without certificate or private key content;
- `GET /open-api/v1/certificates/{cert_name}`: returns details of a single certificate, likewise without sensitive content.

This design ensures that certificate private keys are not frequently transmitted over the network; they are only provided explicitly by an administrator during creation and update.

---

## Setting the Default Certificate

### Role of the Default Certificate

When a client accesses the gateway over HTTPS but the requested domain name (Server Name Indication, SNI) matches no configured certificate, BFE falls back to the **default certificate** to complete the TLS handshake. If no default certificate is set, or the default certificate does not match the listening port, the client will receive a TLS handshake failure or a certificate warning.

Therefore, the default certificate must satisfy:

- There is exactly one globally;
- It is always present in the `CertConf` mapping;
- It cannot be deleted directly; it must first be switched to non-default before deletion.

### Ways to Set the Default Certificate

**Method 1: Specify it directly when creating the certificate**

Set `is_default` to `true` in the request body. If a default certificate already exists, the old default certificate automatically becomes non-default.

**Method 2: Update an existing certificate to default**

Call the dedicated endpoint:

```
PATCH /open-api/v1/certificates/{cert_name}/default
```

This endpoint updates the current default certificate to non-default and sets the target certificate as the new default. The returned data is the same as for a normal certificate detail response.

### Deleting a Non-Default Certificate

Call:

```
DELETE /open-api/v1/certificates/{cert_name}
```

Constraints:

- Only non-default certificates can be deleted;
- After deletion, a default certificate must still remain globally (guaranteed by the system).

---

## Certificate Expiration Management

### Source of the Expiration Time

The `expired_date` field is read-only. AI Gateway API parses it from `cert_file_content` when a certificate is created or updated, in the fixed format `YYYY-MM-DD HH:MM:SS`. Administrators do not need to fill it in manually, and it cannot be modified via OpenAPI.

### Daily Monitoring Recommendations

It is recommended to continuously track certificate status in the following ways:

1. **Call the certificate list endpoint regularly**: iterate over `expired_date` and set alerts for certificates expiring within 30 days, 14 days, and 7 days;
2. **Visual reminders**: in the current release you must use an external alerting system or a self-developed operations dashboard to display soon-to-expire or expired certificates; Dashboard does not yet provide a certificate management page;
3. **Example alert rules**:
   - Certificate remaining validity ≤ 30 days: Info;
   - Certificate remaining validity ≤ 7 days: Warning;
   - Certificate expired: Critical, and consider temporarily switching to a backup certificate.

For production environments with many domains, it is recommended to feed the certificate list endpoint into a monitoring/alerting system or CI/CD pipeline to automate expiration reminders. Some teams combine the ACME protocol (e.g. Let's Encrypt) to automatically request new certificates before expiration and upload them to AI Gateway API via OpenAPI, minimizing manual intervention.

### Renewal and Replacement Workflow

Certificate renewal usually includes the following steps:

1. Request a new certificate from the Certificate Authority (CA) and obtain the new PEM certificate and private key;
2. Update the target certificate in AI Gateway API (the current OpenAPI does this by delete-and-recreate; a PUT for content update may be supported later);
3. Confirm that `expired_date` has been refreshed to the new certificate's expiration time;
4. Wait for the Conf Agent to complete the Data Plane hot reload;
5. Use `curl -v https://<domain>` to verify the new certificate takes effect;
6. Keep the old certificate for at least one TTL period, and clean it up only after confirming there is no anomaly.

> Note: the default certificate must be renewed before it expires, otherwise all HTTPS requests to unmatched domains will be affected.

If HTTPS access becomes abnormal after uploading a new certificate, immediately switch back to the old certificate (if it is still valid), or temporarily set another backup certificate as the default, to avoid prolonged service interruption. After rolling back, also verify access is back to normal via `curl -v` or a browser.

---

## HTTPS Access Verification

### Verifying the TLS Handshake

After completing domain binding and certificate upload, you can verify that HTTPS works using command-line tools:

```bash
curl -v https://api.example.com/v1/chat/completions \
  -H "Authorization: Bearer ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"model":"deepseek-v3","messages":[{"role":"user","content":"hello"}]}'
```

Pay attention to the following in the `-v` output:

- `* Server certificate:` shows the target domain correctly;
- `*  subjectAltName:` includes `api.example.com`;
- `*  issuer:` is the expected CA;
- `*  SSL certificate verify ok.` indicates the certificate chain verification passed.

Besides `curl`, you can also use `openssl s_client` to focus on TLS-layer verification:

```bash
echo | openssl s_client -connect api.example.com:443 -servername api.example.com
```

This command outputs the complete certificate chain, protocol version, cipher suites, and other information, and is suitable when you suspect an incomplete certificate chain or abnormal SNI selection.

### Common Troubleshooting Directions

| Symptom | Possible Cause | How to Investigate |
|------|----------|----------|
| Browser reports the certificate as unsafe | Certificate does not match the accessed domain | Check whether the certificate SAN includes the target domain; check whether the HostTable binds the correct domain |
| TLS handshake failure | No default certificate configured | Call `/certificates` to confirm a certificate with `is_default` true exists |
| Request returns 404 | Domain not bound to the correct product | Check the HostTable and RouteTable in `/configs/tls_conf/server_data_conf` |
| Incomplete certificate chain | Missing intermediate certificates | Ensure the certificate content includes the complete chain (server certificate + intermediate certificates) |

---

## Multi-Domain Configuration

In production environments, an AI gateway often needs to expose multiple domains at the same time, for example:

- `api.example.com`: primary business domain;
- `api-cn.example.com`: mainland China regional domain;
- `internal.example.com`: internal testing domain.

### Mapping Between Multiple Domains and Multiple Certificates

Rainway AI Gateway supports the following two modes:

**Single certificate covering multiple domains**

Request a single wildcard or multi-domain certificate containing multiple SANs (Subject Alternative Names), for example covering both `*.example.com` and `api.example.com`. In this case, you only need to upload one certificate and point multiple domains to the same product in the HostTable.

**Multiple certificates, one-to-one mapping**

Upload an independent certificate for each domain or subdomain, for example `cert_api_example_com`, `cert_api_cn_example_com`. BFE automatically selects the matching certificate based on the client SNI; when there is no match, it falls back to the default certificate.

### Configuration Notes

- The `cert_name` of each certificate must be globally unique; domain-related naming is recommended for easy identification;
- Every certificate should have a description noting the domain, purpose, and issuing CA;
- The default certificate should preferably be the one with the broadest coverage and greatest stability;
- Updating one non-default certificate does not affect HTTPS service for other domains.

### Certificate Selection Order

During the TLS handshake, BFE selects the certificate based on the SNI sent by the client. The selection logic can be summarized as:

1. Look up the certificate matching the SNI in `CertConf` (matched against the SAN/CN in the certificate);
2. If a unique match is found, use that certificate;
3. If no match is found, or the client did not send an SNI, use the default certificate specified by `Default`;
4. If the default certificate also does not exist, the TLS handshake fails.

Therefore, when configuring multiple domains, administrators should ensure the SAN list of each certificate covers all target domains, and keep a reliable default certificate reserved for access scenarios that are not covered.

---

## Complete Configuration Example

The following example shows how to complete domain binding, certificate upload, and HTTPS access verification for an AI product.

### Creating a Certificate

```bash
curl -X POST "http://ai-gateway-api:8183/open-api/v1/certificates" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${ADMIN_TOKEN}" \
  -d '{
    "cert_name": "cert_api_example_com",
    "description": "api.example.com production certificate",
    "is_default": true,
    "cert_file_content": "-----BEGIN CERTIFICATE-----\nMIIDXTCCAkWgAwIBAgIJAKoK/heBjcOuMA0GCSqGSIb3DQEBCwUAMEUxCzAJBgNV\nBAYTAkNOMQ4wDAYDVQQIDAVDaGluYTEQMA4GA1UEBwwHQmVpamluZzEUMBIGA1UE\nCgwLZXhhbXBsZS5jb20xFDASBgNVBAMMC2V4YW1wbGUuY29tMB4XDTE0MDUyNjA3\n...\n-----END CERTIFICATE-----",
    "key_file_content": "-----BEGIN RSA PRIVATE KEY-----\nMIIEpQIBAAKCAQEA3Tz2mvC3D1tRkX7B8kYwQ8u8kYwQ8u8...\n-----END RSA PRIVATE KEY-----"
  }'
```

### BFE Certificate Mapping Configuration

An example of `server_cert_conf.data` exported to BFE by AI Gateway API via InnerAPI:

```json
{
    "Version": "20250101000000",
    "Config": {
        "Default": "cert_api_example_com",
        "CertConf": {
            "cert_api_example_com": {
                "ServerCertFile": "tls_conf_20250101000000/cert_api_example_com/server.crt",
                "ServerKeyFile": "tls_conf_20250101000000/cert_api_example_com/server.key",
                "OcspResponseFile": ""
            }
        }
    }
}
```

Conf Agent writes the certificate files into the `tls_conf_{version}/{cert_name}/` directory and calls BFE's hot reload interface to make them take effect.

### Domain and Routing Configuration

The corresponding HostTable and RouteTable fragments in `server_data_conf` are as follows:

```json
{
    "Version": "20250101000000",
    "HostTable": {
        "Version": "20250101000000",
        "DefaultProduct": "AI_product",
        "Hosts": {
            "host-api-example": ["api.example.com", "api-cn.example.com"]
        },
        "HostTags": {
            "AI_product": ["host-api-example"]
        }
    },
    "RouteTable": {
        "Version": "20250101000000",
        "BasicRule": {},
        "ProductRule": {
            "AI_product": [
                {
                    "Cond": "req_host_in(\"api.example.com\")",
                    "ClusterName": "deepseek-cluster"
                }
            ]
        }
    },
    "ClusterConf": { ... }
}
```

### Verifying the Certificate Takes Effect

After the configuration is pushed and hot reloaded, verify with:

```bash
curl -v https://api.example.com/v1/chat/completions \
  -H "Authorization: Bearer ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"model":"deepseek-v3","messages":[{"role":"user","content":"hello"}]}'
```

Pay attention to the certificate information in the output, for example:

```text
* Server certificate:
*  subject: CN=api.example.com
*  start date: Aug 23 16:02:31 2025 GMT
*  expire date: Aug 23 16:02:31 2026 GMT
*  subjectAltName: host "api.example.com" matched cert's "api.example.com"
*  issuer: C=CN; O=Example CA; CN=Example Intermediate CA
*  SSL certificate verify ok.
```

If you see output like this, domain binding, certificate upload, and HTTPS listening are all in effect.

---

## Chapter Summary

- Domain binding is maintained through AI Gateway API and is ultimately exported via InnerAPI as the HostTable and RouteTable in BFE's `server_data_conf`.
- TLS certificates are uploaded via the OpenAPI `/certificates`; the Control Plane validates the certificate and private key format and pairing, and automatically parses the expiration time.
- There must be exactly one default certificate globally, used for TLS fallback when the SNI does not match; the default certificate cannot be deleted directly.
- It is recommended to establish a certificate expiration monitoring mechanism to complete renewal and verification before expiration, avoiding HTTPS service interruption.
- HTTPS verification can use `curl -v` to check the certificate chain, SAN, Issuer, and other information; common problems are mostly related to domain binding, certificate chain completeness, and missing default certificates.
- Multi-domain scenarios can be implemented with a single certificate containing multiple SANs or with independent certificates per domain; BFE automatically selects the best certificate based on SNI.

---

## References

- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/certificates.md`
- `ai-gateway-api/design-docs/api-define/InnerAPI接口定义/server-cert-conf.md`
- `ai-gateway-api/design-docs/api-define/InnerAPI接口定义/server-data-conf.md`
- `bfe/docs/zh_cn/configuration/tls_conf/server_cert_conf.data.md`
- [Chapter 5: Rainway AI Gateway Architecture and Core Concepts](../design/chapter05-system-architecture.md)
- [Chapter 7: Data Plane Forwarding Design: BFE](../design/chapter07-data-plane-design.md)
