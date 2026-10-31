# Troubleshooting Log

Real problems hit while building the lab, with the cause and the fix.

| # | Symptom | Cause | Fix |
| --- | --- | --- | --- |
| 1 | DC audit events (4720, 4728) not logged; `auditpol` showed "No Auditing" | Audit settings were only in the Default Domain Policy; domain controllers follow the Default Domain Controllers Policy | Set Advanced Audit Policy in the Default Domain Controllers Policy and enabled "force audit policy subcategory settings" |
| 2 | Member server: Group Policy failed, no domain controller found | VM still pointed at the old network; its adapter showed "Not Present" | Added a new adapter on VMnet2, set a static IP with DNS 10.0.0.10, rebooted |
| 3 | WS001 couldn't resolve internet names | DC DNS had no forwarders | `Set-DnsServerForwarder -IPAddress 10.0.0.1, 8.8.8.8` |
| 4 | Sysmon: "Failed to open xml configuration" | Config was saved as a folder, not a file | Deleted the folder and downloaded the file with `Invoke-WebRequest` |
| 5 | Wazuh agent: "Duplicate agent name" | Install command reused another machine's agent name | Reinstalled the agent with the correct `WAZUH_AGENT_NAME` |
| 6 | Domain join worked but rename failed ("directory service is busy") | New computer account still being written | Restarted, then ran `Rename-Computer` separately |
| 7 | `nslookup` returned the DC's old IP (192.168.202.202) | DC had a second adapter on an old network registering in DNS | Disabled the extra adapter, removed stale A records, re-registered DNS |
| 8 | `nslookup` for the lab domain returned a public IP | `.digital` is a real public TLD; the firewall forwarded the query to the internet | Pointed lab clients' DNS at the domain controller |
| 9 | pfSense: "Invalid interface name em1" | VM had only one network adapter | Added a second adapter on VMnet2 |
| 10 | VM booted to "EFI Network... Time out" | Missed the "press any key to boot from CD" prompt | Reset and pressed a key right away |
| 11 | Ubuntu got a 192.168.x.x address | Adapter was on VMware NAT, not the lab network | Moved the adapter to VMnet2 and set a static IP |
| 12 | Wazuh install script printed code instead of running | `curl` run without `-O`, so no file was saved | Downloaded with `curl -O`, then ran the script |
| 13 | Kali `apt update`: Elastic repo "not signed" | Elastic's repo key uses SHA1, which apt now rejects | Disabled the unused Elastic repo instead of weakening signature checks |
| 14 | Host RAM at 98% | Six VMs plus browser and desktop apps on 32 GB | Run 4 VMs at a time, close heavy apps, right-size VM memory |
| 15 | Splunk had no Sysmon events; forwarder log showed `errorCode=5` | Forwarder ran under a restricted account with no permission to read the Sysmon log (error 5 = access denied) | `sc.exe config SplunkForwarder obj= LocalSystem`, then restarted the service |
| 16 | WS001: `ipconfig` showed "Media disconnected" | VM's virtual network adapter was disconnected in VMware | Reconnected the adapter on VMnet2 |
| 17 | Splunk web "refused to connect" on port 8000 | Splunk wasn't running yet | Started Splunk and waited for the web interface message |
| 18 | `sudo -u splunk`: user 'splunk' not found | The Splunk 10.6 package didn't create its service account | `useradd -r -m -s /bin/bash splunk`, then `chown -R splunk:splunk /opt/splunk` |
| 19 | Splunk root disk only 28 GB of 60 GB | Ubuntu installer left half the volume group unallocated | `lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv` |
