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
