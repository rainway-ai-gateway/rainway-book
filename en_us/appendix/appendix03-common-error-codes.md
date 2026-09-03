# Appendix 3: Common Error Codes

The error codes returned by the BFE Data Plane in AI gateway scenarios, along with their response formats, trigger conditions, and troubleshooting advice, are fully maintained by the official BFE documentation. This book does not repeat them here; readers can refer to the authoritative definitions for the v1.8.6 release through the following link.

## Error Code Documentation for v1.8.6

- [BFE AI Gateway Error Codes](https://github.com/bfenetworks/bfe/blob/refs/tags/v1.8.6/docs/zh_cn/sys_design/ai_error_codes.md)

That document covers the following:

- Authentication and admission layer error codes (e.g. `NO_API_KEY`, `INVALID_API_KEY`, `KEY_DISABLED`, `SUBNET_NOT_ALLOWED`, `MODEL_NOT_ALLOWED`)
- Rate limit check layer error codes (e.g. `RPM_LIMIT_EXCEEDED`, `TPM_LIMIT_EXCEEDED`, `CONCURRENCY_LIMIT_EXCEEDED`)
- Quota deduction layer error codes (e.g. `QUOTA_EXHAUSTED`, `QUOTA_EXPIRED`, `INTERNAL_QUOTA_ERROR`)
- Forwarding and protocol adaptation layer error codes (e.g. `PROVIDER_PROTOCOL_MISMATCH`)
- Standard error response body structure and field descriptions
- List of reserved error codes
- Correlation between error codes and access log fields
- Troubleshooting advice by HTTP status code

## Related Chapters in This Book

- [Chapter 16: Security Design](../design/chapter16-security-design.md) introduces the error response body and security audit log fields.
- [Chapter 18: Dashboard Basics](../operation/chapter18-dashboard-basics.md) introduces how to interpret common Dashboard error messages.
- [Chapter 21: API-Key and Quota Configuration](../operation/chapter21-apikey-and-quota-config.md) introduces troubleshooting methods for issues such as quota exhaustion and rate limit triggers.
