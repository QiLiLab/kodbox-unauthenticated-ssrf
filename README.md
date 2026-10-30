# kodbox public-share SSRF with response disclosure

[简体中文](README.zh-CN.md) · [Local reproduction](POC.md)

This repository records a reported server-side request forgery (SSRF) in the kodbox public-share interface. A valid, accessible public share link is required, but the request does not require an account. The supplied report documents a local reproduction on `kodcloud/kodbox:1.68.10`; the relevant URL-accepting branch is also present in the upstream `1.70.03` release source. Runtime behavior on `1.70.03` has not been independently reproduced here.

## Summary

`explorer.share.fileGet` resolves the requested `path` through `share.class.php::parsePath()`. The function returns an allowed URL before applying the checks that keep ordinary paths inside the shared root. The URL then reaches the file-reading pipeline, where kodbox fetches it server-side and returns the body in the JSON `data.content` field. This lets an anonymous visitor who knows a valid public share ID request content reachable by the kodbox server, including internal HTTP services.

The same URL path is reportedly accepted by `fileOut` / `fileOutBy` and can produce an unauthenticated external redirect. That secondary behavior is described in the supplied report; it is separate from the response-disclosure finding.

| Item | Detail |
| --- | --- |
| Product | [kalcaddle/kodbox](https://github.com/kalcaddle/kodbox) |
| Observed runtime version | Official Docker image `kodcloud/kodbox:1.68.10`, according to the supplied report |
| Source version checked | Upstream `1.70.03` release commit [`d5b517b`](https://github.com/kalcaddle/kodbox/commit/d5b517b) |
| Weakness | CWE-918, server-side request forgery |
| Access requirement | A valid public share without a password, expiry, or preview restriction; no authenticated account |
| Reported result | Full body of a container-internal test service returned in `data.content` |
| Suggested CVSS 3.1 vector | `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:N/A:N` (8.6); provisional assessment |

## Root cause

At the [`1.70.03` source snapshot](https://github.com/kalcaddle/kodbox/blob/d5b517b/app/controller/explorer/share.class.php#L311-L317), `parsePath()` starts with:

```php
private function parsePath($path,$allowNotExist=false){
    if(request_url_safe($path)) return $path;//压缩包支持;
    $rootSource = $this->share['sourceInfo']['path'];
    // Share-root validation follows.
}
```

[`fileGet()` passes the request path through this function](https://github.com/kalcaddle/kodbox/blob/d5b517b/app/controller/explorer/share.class.php#L460-L465). [`request_url_safe()`](https://github.com/kalcaddle/kodbox/blob/d5b517b/app/function/web.function.php#L760-L779) allows `http`, `https`, and `ftp` URLs and has no mandatory check for loopback, private, or link-local destinations. An optional site-specific `request_url_safe_self` hook can be defined, but it is not the default protection.

The supplied report traces the accepted URL into `explorer.editor.fileGet` and `PathDriverUrl`, then to the server-side request helper. This downstream path and the response example were not independently re-run while preparing this repository.

## Reported local evidence

The report used a public share on an isolated Docker instance and a test HTTP service bound to `127.0.0.1:9999` inside the container. The port was not mapped to the host. An anonymous request with the test-service URL as `path` reportedly returned:

```json
{
  "code": true,
  "data": {
    "size": 18,
    "content": "INTERNAL-ONLY-DATA"
  }
}
```

The report also records two control cases: a `file://` path was rejected, and an invalid share ID returned “share does not exist.” These support the stated scheme check and public-share prerequisite. [POC.md](POC.md) preserves a local-only reproduction sequence without real session tokens or a reusable share hash.

## Impact and limits

The direct impact is disclosure of responses from services the kodbox server can reach. Possible targets include internal web applications and cloud metadata endpoints when the hosting environment permits those connections. Exposure of credentials depends on the specific environment and service; this report does not establish that any third-party instance or cloud credential was accessed.

The share must be public and accessible. The finding is therefore unauthenticated **after** a valid share ID is known, rather than an endpoint that works without any share context.

## Related public advisory

[CVE-2026-5618 / GHSA-582g-j67j-5q79](https://github.com/advisories/GHSA-582g-j67j-5q79) describes SSRF in kodbox's `shareMake/shareCheck` component through `siteFrom/siteTo`, affecting versions up to 1.64 according to that advisory. This report concerns `explorer.share.fileGet` and the `path` parameter, with a reported local reproduction on 1.68.10. The available evidence shows a different entry point; whether maintainers consider the findings related or duplicates requires their review. No new CVE is claimed here.

## Recommended remediation

1. Reject arbitrary URLs in `share.class.php::parsePath()` for public-share file reads, or bind any supported URL to an explicitly shared object.
2. Enforce destination restrictions in the common URL-fetch path, including resolved addresses and redirects. Cover loopback, RFC 1918, link-local, IPv6 local ranges, and hostnames that resolve into them.
3. Prevent URL paths from becoming external `Location` headers in `fileOut` / `fileOutBy`.
4. Add regression cases for anonymous public-share requests to internal and external URLs.

## Disclosure status

This repository is public and contains a sanitized technical record. Publishing it does not notify the kodbox maintainers or submit a private vulnerability report to their repository.
