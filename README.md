# Wazuh-for-MikroTik

MikroTik RouterOS Decoders & Detection Rules (Wazuh/OSSEC)

This repository provides a small **custom decoder** and **correlation ruleset** to parse and alert on **MikroTik RouterOS** logs in **Wazuh / OSSEC**.

  Files

- `decoder.xml` — XML **decoders** for MikroTik RouterOS log messages, including:
  - VPN PPP login/logout
  - Admin login/logout and login failures (with source IP and protocol)
  - Firewall drops (SRC/DST/PROTO and optional DPT)
  - Interface link up/down
  - IPsec repeated error messages

- `rules.xml` — XML **rules** and **correlations**, including:
  - VPN login/logout + reconnect anomaly correlation
  - PPP authentication failures + brute-force correlation
  - IPsec errors / repeated error correlation
  - Firewall DROP INPUT / DROP FORWARD + DoS/flood correlation (many drops from same source)
  - DHCP lease assigned, DHCP conflict + frequent conflict correlation
  - Admin login/logout, admin login failed + brute-force correlation
  - Reboot events
  - Link up/down + link flapping correlation

  

  Requirements

- Wazuh Manager (or OSSEC) with custom decoders/rules enabled.
- MikroTik logs delivered to the manager/agent (e.g., via syslog).

  

  Installation (Wazuh)

1. **Install the decoder**
   Copy `decoder.xml` to your decoder directory, for example:
   - `/var/ossec/etc/decoders/decoder.xml`

2. **Install the rules**
   Copy `rules.xml` to your rules directory, for example:
   - `/var/ossec/etc/rules/rules.xml`

3. **Replace placeholder variables**
   `rules.xml` contains placeholders such as:
   - `LEVEL_*` (rule severity levels)
   - `FREQ_*`, `TF_*`, `IGN_*` (correlation thresholds: frequency, timeframe, ignore/suppression)

   These placeholders are intended for templating (envsubst/Jinja/Ansible) or manual replacement.

   **Manual guidance**
   - `LEVEL_*` → severity level (integer)
   - `FREQ_*` → frequency threshold (count)
   - `TF_*` → timeframe (seconds)
   - `IGN_*` → ignore/suppression window (seconds)

   **Example using envsubst**

   export LEVEL_VPN_LOGIN=3
   export LEVEL_VPN_LOGOUT=2
   export LEVEL_VPN_RECONNECT=5
   export FREQ_VPN_RECONNECT=3
   export TF_VPN_RECONNECT=60
   export IGN_VPN_RECONNECT=300

   envsubst < rules.xml > rules.rendered.xml

4. **Restart Wazuh manager**

   sudo systemctl restart wazuh-manager
  

  Log Format Expectations

The decoder expects RouterOS lines that start with **comma-separated tags**, for example:

* VPN login/logout:

  * `l2tp,ppp,info,account user logged in`
  * `l2tp,ppp,info,account user logged out`

* Admin activity:

  * `system,info,account user admin logged in from 203.0.113.10 via ssh`
  * `system,info,account user admin login failed from 203.0.113.10 via winbox`

* Firewall drops (fields may vary by your log prefix):

  * `firewall,info DROP INPUT ... SRC=203.0.113.10 DST=192.0.2.5 PROTO=TCP DPT=22`

* Interface state:

  * `interface,info ether1 link down`
  * `interface,info ether1 link up`

* IPsec repeated error:

  * `ipsec,error message repeated 10 times: [ 203.0.113.10 <error text> ]`

If your RouterOS adds timestamps/hostnames before the tags, adjust `prematch` / `regex` patterns accordingly.

  

  Testing (Recommended)

Validate decoding and rule matching with Wazuh logtest:


sudo /var/ossec/bin/wazuh-logtest


Paste a sample log line and confirm:

* the intended decoder is selected
* expected fields are extracted (e.g., `srcip`, `dstip`, `dstport`, `protocol`, `user`)
* the correct rule triggers

  

  Notes / Customization

* Some rule descriptions reference variables like `$(dstuser)`. Depending on your decoder schema/version, you may want to use `$(user)` instead.
* Correlation thresholds are intentionally left as placeholders—tune them to your environment to reduce noise.
* If you plan to keep this repository public, avoid committing environment-specific details (public IPs, internal subnets, hostnames, real usernames).

  

  Disclaimer

Provided “as is”. Review, test, and adapt before production use. Not affiliated with MikroTik.


