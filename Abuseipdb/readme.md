# AbuseIPDB Integration

This integration allows Wazuh to query the AbuseIPDB database and generate alerts based on the **Confidence Score** — a numeric value representing the likelihood that a given IP address is malicious.

---

## Step 1: Create the Integration Script

Create the custom integration script file at the following path:

```bash
nano /var/ossec/integrations/custom-abuseipdb.py
```

---

## Step 2: Set Permissions

Run the following commands to ensure the Wazuh manager can execute the script and the script can write its results to the output log file.

```bash
# Script (owned by root:wazuh)
chown root:wazuh /var/ossec/integrations/custom-abuseipdb.py
chmod 750 /var/ossec/integrations/custom-abuseipdb.py

# Log file (owned by wazuh:wazuh to allow the script to write)
touch /var/ossec/logs/external/abuseipdb.log
chown wazuh:wazuh /var/ossec/logs/external/abuseipdb.log
chmod 660 /var/ossec/logs/external/abuseipdb.log
```

---

## Step 3: Configure ossec.conf

Add the integration block and the log monitoring entry to the main `ossec.conf` file.

```xml
<!-- AbuseIPDB Integration -->
<integration>
  <name>custom-abuseipdb.py</name>
  <group>web,attack,accesslog,ids</group>
  <alert_format>json</alert_format>
</integration>

<localfile>
  <log_format>json</log_format>
  <location>/var/ossec/logs/external/abuseipdb.log</location>
</localfile>
```

---

## Step 4: Create the Detection Rules

Add the following rules to `local_rules.xml`. Three rules are defined: a base event rule, a warning for suspicious IPs (score 1–49), and a critical alert for confirmed malicious IPs (score 50–100).

```xml
<!--##############################-->
<!--## AbuseIPDB Integration   ##-->
<!--##############################-->

<group name="abuseipdb_intel,">
  <rule id="100013" level="0">
    <decoded_as>json</decoded_as>
    <field name="integration">abuseipdb</field>
    <description>AbuseIPDB reputation events</description>
  </rule>

  <rule id="100014" level="5">
    <if_sid>100013</if_sid>
    <field name="abuse_data.score" type="pcre2">^[1-9]|[1-4][0-9]$</field>
    <description>AbuseIPDB: Suspicious IP detected ($(abuse_data.ip)) - Score: $(abuse_data.score)</description>
  </rule>

  <rule id="100015" level="12">
    <if_sid>100013</if_sid>
    <field name="abuse_data.score" type="pcre2">^([5-9][0-9]|100)$</field>
    <description>AbuseIPDB: MALICIOUS IP confirmed ($(abuse_data.ip)) - Score: $(abuse_data.score)</description>
    <mitre>
      <id>T1583</id>
    </mitre>
    <group>attack,threat_intel,</group>
  </rule>
</group>
```

---

## Step 5: Generate a Test Event

To verify the integration is working, write a synthetic log entry using an IP address known to have a high AbuseIPDB score.

```bash
echo '193.163.125.138 - - [06/Mar/2026:17:00:00 -0300] "GET /admin HTTP/1.1" 401 512 "-" "Mozilla/5.0"' >> /var/ossec/logs/external/test.log
```

> 💡 If the integration is correctly configured, this entry should trigger the AbuseIPDB lookup. Depending on the IP's current score, either rule `100014` or `100015` will fire.