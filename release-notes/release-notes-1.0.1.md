# Amazon::Signature4::Lite 1.0.1 Release Notes

## Overview

1.0.1 fixes three signing correctness bugs discovered when `Amazon::API`
migrated from `AWS::Signature4` to `Amazon::Signature4::Lite` as its signing
dependency (2.3.0). All three caused `SignatureDoesNotMatch` errors against
live AWS services.

---

## Bug Fixes

### Empty path in canonical request

URLs with no path component (e.g. `https://sts.amazonaws.com`) return an
empty string from `URI->path`, not `undef`. The previous `$path //= '/'`
guard only fires for `undef`, so bare-hostname URLs produced an empty path
line in the canonical request instead of the required `/`. Fixed by changing
to `$path = $path || '/'`.

### Query string double-encoding

Pre-encoded query string values (e.g.
`EventSourceArn=arn%3Aaws%3Asqs%3A...`) were being double-encoded -
`%3A` became `%253A` - because the raw query string was split on `&`/`=`
and then passed directly to `uri_escape_utf8` without first decoding the
existing percent-encoding. Fixed by calling `uri_unescape` before
`uri_escape_utf8` on each key and value, normalizing already-encoded values
before re-encoding. This affected any REST service with complex values in
query parameters (Lambda `ListEventSourceMappings`, etc.).

### `add_sha256_header` option

`sign()` now accepts an `add_sha256_header` argument (default `1`, preserving
backward compatibility for S3 which requires `x-amz-content-sha256` as a
signed header). When set to `0`, `x-amz-content-sha256` is not added to the
request headers or `SignedHeaders` - matching the behavior of `AWS::Signature4`
for query-protocol and REST-JSON services (STS, SNS, Lambda, etc.) that do
not expect this header to be signed.

Note: the payload hash is always computed for the canonical request regardless
of this setting - only the request header itself is conditional.

---

## Upgrade Notes

Callers using `Amazon::Signature4::Lite` directly for S3 are unaffected -
`add_sha256_header` defaults to `1`. Callers using it for non-S3 services
should pass `add_sha256_header => 0` to `sign()` to match AWS's expected
`SignedHeaders` for those services.
