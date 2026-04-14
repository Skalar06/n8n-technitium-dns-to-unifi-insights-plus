# Technitium DNS -> UniFi Log Insights DNS

n8n workflow for receiving Technitium DNS query logs via webhook, transforming them into an RFC3164 / dnsmasq-like syslog format, and forwarding them to UniFi Log Insights via UDP syslog.

## Purpose

This workflow acts as a simple bridge between:

- **Technitium DNS**
- **n8n**
- **UniFi Log Insights**

It receives DNS log entries as JSON, filters unwanted records, and converts them into syslog lines in a `dnsmasq`-style format so UniFi Log Insights can recognize and display them more naturally.

## How it works

The workflow does the following:

1. Receives HTTP POST requests through an n8n webhook
2. Validates a shared token in the `x-log-exporter-token` header
3. Parses the incoming DNS log data
4. Filters selected record types and defined client IPs
5. Converts the data into RFC3164 syslog format
6. Sends the resulting syslog line via UDP to `unifi-log-insight:514`

Example output:

```text
<14>Apr 13 22:07:05 technitium-dns dnsmasq[1]: query[A] cloudaccess.svc.ui.com from 10.0.0.50 proto=Udp rcode=NoError rtype=Recursive
```

## Requirements

- a running **n8n** instance
- a reachable **UniFi Log Insights** syslog receiver
- Technitium DNS or another sender that can post compatible JSON payloads to the webhook
- network connectivity from n8n to the syslog target on UDP port **514**

## Import into n8n

1. Import the workflow JSON into n8n
2. Open the workflow
3. Adjust the following values:
   - webhook path
   - shared token
   - list of ignored client IPs
   - syslog target hostname
4. Activate the workflow

## Important configuration changes

### 1. Set your shared token

The `If` node currently contains a placeholder:

```text
CHANGE_ME_SHARED_TOKEN
```

Replace it with your own randomly generated shared secret.

The sender must use the same value in the HTTP header:

```text
x-log-exporter-token: YOUR_TOKEN
```

### 2. Adjust the syslog destination host

The `Execute Command` node currently sends to:

```text
unifi-log-insight
```

Change this to match your environment if needed.

### 3. Adjust blocked client IPs

The Code node currently contains this example list:

```javascript
const blockedClientIps = new Set([
  "192.0.2.10",
  "192.0.2.11",
  "198.51.100.10",
  "172.16.0.1",
]);
```

These are placeholder values and should be replaced with the real client IPs you want to ignore.

### 4. Adjust filtered DNS record types

By default, these record types are filtered out:

```javascript
const blockedTypes = new Set(["SOA", "IXFR"]);
```

You can extend or reduce this list as needed.

## Expected input format

The workflow expects JSON data, for example:

```json
{
  "timestamp": "2026-04-13T22:07:05.486Z",
  "clientIp": "10.0.0.50",
  "protocol": "Udp",
  "question": {
    "questionClass": "IN",
    "questionName": "cloudaccess.svc.ui.com",
    "questionType": "A"
  },
  "responseCode": "NoError",
  "responseType": "Recursive"
}
```

Both single objects and arrays are supported.

The workflow also attempts to parse JSON stored inside fields such as `RenderedMessage` or `MessageTemplate`.

## Required header

The webhook expects this header:

```text
x-log-exporter-token: YOUR_TOKEN
```

If the header is missing or the value does not match, the workflow responds with HTTP **403**.

## Webhook responses

### Success

```json
{"ok":true}
```

HTTP status: `200`

### Unauthorized

If the token is missing or invalid:

HTTP status: `403`

## Security notes

This workflow is intended for use in **trusted internal environments**.

Recommendations:

- do not expose the webhook publicly without protection
- use a long, random shared token
- rotate the token regularly
- use reverse proxy restrictions or IP allowlists in addition
- keep n8n accessible only within controlled networks or properly secured

## Notes about shell execution

In this example, forwarding to UniFi Log Insights is done with an `Execute Command` node using `nc`:

```sh
printf "%s\n" "<syslog-line>" | nc -u -w1 unifi-log-insight 514
```

This is simple and practical, but it requires `nc` to be available in the n8n container or on the n8n host.

Depending on your setup, another forwarding method may be more appropriate.

## Known limitations

- the syslog format is intentionally simplified
- DNS response details are not fully represented
- timestamp formatting is fixed to `Europe/Berlin`
- the workflow is focused on query-log forwarding, not full DNS audit telemetry

## Use cases

Suitable for:

- DNS query visibility in UniFi Log Insights
- basic correlation of DNS requests with other infrastructure logs
- small to medium homelab or SMB environments
- a quick bridge without building dedicated middleware

## Technitium Log Exporter App configuration

This workflow expects DNS query logs to be sent by the Technitium **Log Exporter App** using **HTTP POST**. Technitium supports exporting query logs to HTTP/HTTPS POST and Syslog sinks, and current versions support custom HTTP headers such as `x-log-exporter-token`. Do **not** publish your real token in the repository. Use placeholders in documentation and a unique secret in production.

### Example configuration

Configure the Technitium Log Exporter App approximately like this:

- **Export / Sink type:** HTTP or HTTPS POST
- **Webhook URL:** `https://YOUR-N8N-DOMAIN/webhook/technitium-dns-log-export`
- **HTTP method:** `POST`
- **Content-Type:** `application/json`
- **Custom header:** `x-log-exporter-token: YOUR_TOKEN`

### Expected payload

The n8n workflow accepts a JSON object like this:

```json
{
  "timestamp": "2026-04-13T22:07:05.486Z",
  "clientIp": "10.0.0.50",
  "protocol": "Udp",
  "question": {
    "questionClass": "IN",
    "questionName": "cloudaccess.svc.ui.com",
    "questionType": "A"
  },
  "responseCode": "NoError",
  "responseType": "Recursive"
}
```

The workflow also supports arrays of entries and will additionally try to parse JSON embedded in fields such as `RenderedMessage` or `MessageTemplate`.

### Header requirement

The webhook validates this header:

```text
x-log-exporter-token: YOUR_TOKEN
```

If the header is missing or does not match the configured shared secret, the workflow will reject the request with HTTP `403`.

### Notes

- UI labels in Technitium may vary slightly by version.
- Keep the real token out of GitHub and use a placeholder such as `YOUR_TOKEN` in all examples.
- If your Technitium instance sends a slightly different JSON structure, adapt the n8n Code node accordingly.
- Test with a single DNS query first before enabling permanent export.


## License

Free to use and modify at your own risk.
