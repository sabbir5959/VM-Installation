# Wazuh Custom Rules Documentation for Company Dashboard

## Overview

Ei README-tta amar VM-e Apache + Wazuh diye test kora custom rules-er jonno. Ei rules-gulo amra company-er Wazuh environment-e reuse kore Palo Alto firewall er logs theke security monitoring and dashboard banate pari.

Amar main goal holo:
- Apache lab rules ke Wazuh-e validate kora
- Palo Alto theke log collect kora
- Custom decoder/rule diye alert generate kora
- Wazuh Discover, dashboard, visualization-e show kora

## Detection Flow

```text
Log Source
   |
   v
Wazuh Agent / Syslog Forwarder
   |
   v
Decoder (Built-in or Custom)
   |
   v
Rule
   |
   v
Alert
   |
   v
Dashboard / Discover
```

## Lab Rules (Apache-Based Test Cases)

### 1. PHP Code Injection

- Built-in decoder: web-accesslog
- Rule ID: 100500
- MITRE: T1190

Rule:

```xml
<rule id="100500" level="12">
  <if_group>accesslog</if_group>
  <match>php://</match>
  <description>Possible PHP Code Injection Attempt</description>
  <group>web_attack,php_injection,</group>
  <mitre><id>T1190</id></mitre>
</rule>
```

Test:

```bash
curl "http://localhost/index.php?page=php://input"
```

### 2. HTML Injection

- Built-in decoder: web-accesslog
- Rule ID: 100620
- MITRE: T1190

Rule:

```xml
<rule id="100620" level="10">
  <if_group>accesslog</if_group>
  <match>Injected</match>
  <description>Possible HTML Injection Attempt</description>
  <group>web_attack,html_injection,</group>
  <mitre><id>T1190</id></mitre>
</rule>
```

Test:

```bash
curl "http://localhost/index.php?name=<h1>Injected</h1>"
```

### 3. XML Bomb

Ei case-ta custom decoder lage karon application log built-in decoder diye match hocche na.

Decoder:

```xml
<decoder name="xml_bomb">
    <program_name>^myapp$</program_name>
    <prematch>ERROR XML entity expansion detected</prematch>
    <regex>^(ERROR XML entity expansion detected while parsing request.*)$</regex>
    <order>message</order>
</decoder>
```

Rule:

```xml
<rule id="100610" level="10">
    <decoded_as>xml_bomb</decoded_as>
    <match>XML entity expansion detected</match>
    <description>Possible XML Bomb / XML Entity Expansion Attempt</description>
    <group>web_attack,xml_bomb,</group>
    <mitre><id>T1190</id></mitre>
</rule>
```

Test:

```bash
logger -t myapp "ERROR XML entity expansion detected while parsing request"
```

## Company Use Case: Palo Alto to Wazuh

Company-e jodi Palo Alto firewall theke log collect kora hoy, tahole same workflow follow kora jay:

1. Palo Alto theke log Wazuh-e forward kora
   - syslog / CEF / agent path diye
2. Wazuh-e decoder match hocche kina check kora
3. Jodi built-in decoder na pay, custom decoder create kora
4. Rule build kora
5. Alert Wazuh Discover-e dekhano
6. Dashboard-e widget banano

## How to Adapt Apache Rules for Palo Alto Logs

Apache rules er logic ke Palo Alto logs-e apply korar jonno amra usually ei fields gulor upor match kori:
- URL / URI
- HTTP request
- Threat name
- Source IP
- Destination IP
- Message / Description
- Severity

Jodi Palo Alto er log message-e ei fields gulo thake, tahole same rule structure use kora jay.

## Example Palo Alto Rule Template

```xml
<decoder name="paloalto_threat">
    <program_name>^PA-Firewall$</program_name>
    <prematch>THREAT</prematch>
    <regex>^(.*)$</regex>
    <order>message</order>
</decoder>

<rule id="100700" level="10">
    <decoded_as>paloalto_threat</decoded_as>
    <match>THREAT</match>
    <description>Palo Alto threat event detected</description>
    <group>network,firewall,threat,</group>
    <mitre><id>T1190</id></mitre>
</rule>
```

> Note: Ei example template hocche generic pattern. Amar company-er actual Palo Alto log format er basis-e decoder er regex adjust korte hobe.

## When to Create a Custom Decoder

Jodi `wazuh-logtest`-er Phase 2-e eta dekhay:

```text
No decoder matched
```

tahole custom decoder create kora lagbe.

Jodi already decoder match kore, tahole only custom rule create kora jabe.

## Wazuh Dashboard Ideas for Company

Palo Alto logs theke dashboard-e ei widgets rakha jete pare:
- Alert count over time
- Top source IPs
- Top destination IPs
- Top threat categories
- Severity distribution
- Blocked URL / suspicious web activity
- Firewall event trend by hour/day

## Validation Checklist

- Palo Alto log Wazuh-e reach kore
- Decoder match kore
- Rule trigger hoy
- Alert Discover-e dekhay
- Dashboard-e widget show kore
- MITRE mapping thake

## Migration Strategy

1. Sample Palo Alto log identify kora
2. Wazuh-e decoder check kora
3. Jodi decoder na thake, custom decoder create kora
4. Rule logic reuse kora
5. `wazuh-logtest` diye validate kora
6. Alert Discover and dashboard-e verify kora
