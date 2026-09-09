# Wazuh AlienVault OTX Integration

This guide shows how to integrate AlienVault OTX with Wazuh 4.14.6 so alerts can be enriched with external threat intelligence.

When a Wazuh alert contains a source IP address (`srcip`), a custom integration script queries AlienVault OTX. If the IP is found in threat intelligence, the script writes an enriched JSON event to `otx.log`, which Wazuh ingests and displays in the dashboard.

## Workflow

```text
Log Source
	│
	▼
Wazuh Rule Match
	│
	▼
custom-otx.py
	│
	▼
AlienVault OTX API
	│
	▼
Threat Intelligence Result
	│
	▼
otx.log (JSON)
	│
	▼
Wazuh Dashboard
```

## Prerequisites

- Wazuh Manager 4.14.6
- Internet connectivity
- AlienVault OTX API key
- Root or sudo privileges

## 1. Install Required Python Packages

Use Wazuh's built-in Python environment.

```bash
sudo /var/ossec/framework/python/bin/python3 -m pip install OTXv2 requests
```

Verify the installation:

```bash
sudo /var/ossec/framework/python/bin/python3 -c "from OTXv2 import OTXv2; import requests; print('OTX OK')"
```

Expected output:

```text
OTX OK
```

## 2. Create the API Key File

Create a secure file for storing the OTX API key.

```bash
sudo nano /var/ossec/integrations/otx.key
```

File content:

```text
YOUR_OTX_API_KEY
```

Set permissions:

```bash
sudo chown root:wazuh /var/ossec/integrations/otx.key
sudo chmod 640 /var/ossec/integrations/otx.key
```

Verify:

```bash
ls -l /var/ossec/integrations/otx.key
```

Expected:

```text
-rw-r----- root wazuh
```

## 3. Create the Integration Script

Create the script below at `/var/ossec/integrations/custom-otx.py`.

```python
#!/var/ossec/framework/python/bin/python3

import ipaddress
import json
import sys
import time

from OTXv2 import OTXv2, IndicatorTypes

API_KEY_FILE = "/var/ossec/integrations/otx.key"
OTX_LOG_FILE = "/var/ossec/logs/external/otx.log"
INTEGRATIONS_LOG = "/var/ossec/logs/integrations.log"

with open(API_KEY_FILE, "r", encoding="utf-8") as file_handle:
    API_KEY = file_handle.read().strip()


def log_msg(message):
    timestamp = time.strftime("%Y-%m-%d %H:%M:%S")
    with open(INTEGRATIONS_LOG, "a", encoding="utf-8") as file_handle:
        file_handle.write(f"{timestamp} - OTX_Integrator: {message}\n")


def write_otx_event(data):
    with open(OTX_LOG_FILE, "a", encoding="utf-8") as file_handle:
        file_handle.write(json.dumps(data, ensure_ascii=False) + "\n")


def is_valid_ipv4(ip):
    try:
        return isinstance(ipaddress.ip_address(ip), ipaddress.IPv4Address)
    except ValueError:
        return False


def check_ip(otx, alert, ip, ip_type):
    log_msg(f"INFO - Checking {ip_type}: {ip}")

    result = otx.get_indicator_details_full(IndicatorTypes.IPv4, ip)
    pulses = result.get("general", {}).get("pulse_info", {}).get("pulses", [])

    if pulses:
        output = {
            "integration": "alienvault_otx",
            "ip_type": ip_type,
            "otx_response": {
                "indicator": ip,
                "threat_found": "yes",
                "pulses_count": len(pulses),
                "pulse_names": [pulse.get("name") for pulse in pulses[:3]],
                "original_alert_id": alert.get("id")
            }
        }

        write_otx_event(output)
        log_msg(f"SUCCESS - Threat found for {ip_type}: {ip}")
    else:
        log_msg(f"INFO - {ip_type} {ip} is clean.")


def main():
    if len(sys.argv) < 2:
        sys.exit(1)

    try:
        with open(sys.argv[1], encoding="utf-8") as file_handle:
            alert = json.load(file_handle)

        data = alert.get("data", {})

        srcip = data.get("srcip") or alert.get("srcip")
        dstip = data.get("dstip") or alert.get("dstip")

        ips_to_check = []

        if srcip and is_valid_ipv4(srcip):
            ips_to_check.append(("srcip", srcip))

        if dstip and is_valid_ipv4(dstip) and dstip != srcip:
            ips_to_check.append(("dstip", dstip))

        if not ips_to_check:
            log_msg("DEBUG - No valid IPv4 srcip/dstip found.")
            sys.exit(0)

        otx = OTXv2(API_KEY)

        for ip_type, ip in ips_to_check:
            try:
                check_ip(otx, alert, ip, ip_type)
            except Exception as exception:
                log_msg(f"ERROR - Failed to check {ip_type} {ip}: {exception}")

    except Exception as exception:
        log_msg(f"ERROR - {exception}")


if __name__ == "__main__":
    main()
```

Set permissions:

```bash
sudo chown root:wazuh /var/ossec/integrations/custom-otx.py
sudo chmod 750 /var/ossec/integrations/custom-otx.py
```

Verify:

```bash
ls -l /var/ossec/integrations/custom-otx.py
```

Expected:

```text
-rwxr-x--- root wazuh
```

## 4. Create the External Log Directory

```bash
sudo mkdir -p /var/ossec/logs/external
sudo touch /var/ossec/logs/external/otx.log
sudo chown root:wazuh /var/ossec/logs/external/otx.log
sudo chmod 664 /var/ossec/logs/external/otx.log
```

Avoid using `chmod 777` in production.

## 5. Configure `ossec.conf`

Edit `/var/ossec/etc/ossec.conf` and add the following block before the closing `</ossec_config>` tag.

### Integration

```xml
<integration>
  <name>custom-otx.py</name>
  <group>attack,syscheck,ids,web,accesslog,paloalto,firewall</group>
  <alert_format>json</alert_format>
</integration>
```

### Monitor OTX Output

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/ossec/logs/external/otx.log</location>
</localfile>
```

## 6. Restart Wazuh

```bash
sudo /var/ossec/bin/wazuh-control restart
```

Verify that the integration loaded:

```bash
sudo grep -Ei "custom-otx|integratord" /var/ossec/logs/ossec.log
```

Expected:

```text
Enabling integration for: 'custom-otx.py'
```

## Testing the Integration

### Create a Test Log

If you are using an Apache-style test file:

```bash
echo '141.98.10.210 - - [06/Mar/2026:17:00:00 -0300] "GET /admin HTTP/1.1" 401 512 "-" "Mozilla/5.0"' | sudo tee -a /var/ossec/logs/external/test.log
```

### Monitor the Integration Log

In one terminal:

```bash
sudo tail -f /var/ossec/logs/integrations.log
```

Expected output:

```text
INFO - Checking 141.98.10.210
SUCCESS - Threat found for 141.98.10.210
```

### Monitor the OTX Output Log

In another terminal:

```bash
sudo tail -f /var/ossec/logs/external/otx.log
```

Expected output:

```json
{
  "integration": "alienvault_otx",
  "otx_response": {
	"indicator": "141.98.10.210",
	"threat_found": "yes",
	"pulses_count": 50
  }
}
```

## Log Explanation

### `integrations.log`

Location:

```text
/var/ossec/logs/integrations.log
```

Purpose:

- Script execution status
- Debug information
- Success messages
- Error messages

Examples:

```text
INFO - Checking 141.98.10.210
SUCCESS - Threat found
DEBUG - No srcip found.
ERROR - API failure
```

### `otx.log`

Location:

```text
/var/ossec/logs/external/otx.log
```

Purpose:

- Final enriched threat intelligence event
- Parsed by the Wazuh Dashboard

Example:

```json
{
  "integration": "alienvault_otx",
  "otx_response": {
	"indicator": "141.98.10.210",
	"threat_found": "yes",
	"pulses_count": 50,
	"pulse_names": [
	  "IOC Records",
	  "Botnet List",
	  "Honeypot Visitors"
	]
  }
}
```

## Troubleshooting

### Integration Loaded

```bash
sudo grep -Ei "custom-otx|integratord" /var/ossec/logs/ossec.log
```

Expected:

```text
Enabling integration for: 'custom-otx.py'
```

### Live Debug

```bash
sudo tail -f /var/ossec/logs/integrations.log
```

### OTX Events

```bash
sudo tail -f /var/ossec/logs/external/otx.log
```

### Common Issues

| Problem | Solution |
| --- | --- |
| No `srcip` found | Ensure the alert contains a source IP. |
| API error | Verify `otx.key` and internet connectivity. |
| Script not running | Check script permissions are `750`. |
| No dashboard event | Confirm `otx.log` is configured as a `localfile`. |

## Security Best Practices

- Store the API key in `/var/ossec/integrations/otx.key`.
- Keep permissions at `640`.
- Never hardcode secrets inside the Python script.
- Avoid `chmod 777` in production.
- Restart Wazuh after configuration changes.
- Monitor `integrations.log` regularly for errors.

## Validation Checklist

- OTX Python package installed
- API key stored in `otx.key`
- `custom-otx.py` created
- Script permissions set to `750`
- `otx.log` created
- `ossec.conf` updated
- Wazuh restarted
- `integrations.log` shows successful execution
- `otx.log` receives enriched JSON events
- OTX events appear in the Wazuh Dashboard
