# Wazuh File Integrity Monitoring (FIM): Manager on Ubuntu, Agent on Windows 11

A hands-on lab deploying [Wazuh](https://wazuh.com/) (v4.14.9) with an Ubuntu VM as the manager and a Windows 11 host as the monitored endpoint, then configuring File Integrity Monitoring (FIM) to detect file changes in a specific directory in real time.

> **Disclaimer:** This project was conducted in a personal, authorized home lab for educational and defensive security purposes.

## Objectives

- Deploy a Wazuh server (manager, indexer, dashboard) on Ubuntu
- Enroll a Windows 11 host as a Wazuh agent
- Configure FIM on a chosen directory by editing the agent's `ossec.conf`
- Generate file events and verify they are detected, classified, and displayed in the dashboard

## Lab Architecture

| Role | System | Details |
|---|---|---|
| Wazuh manager + indexer + dashboard | Ubuntu 64-bit VM (VMware) | Wazuh v4.14.9, dashboard on `https://<manager-ip>:443` |
| Wazuh agent | Windows 11 Home (physical host) | Agent name `WinddowsHost` (ID 001) |
| Network | Bridged (192.168.x.x) | VM and host on the same LAN |

## Tools

Wazuh (manager, indexer, dashboard, agent) · Ubuntu · Windows 11 · VMware · Bridged networking · `ossec.conf`

## Implementation

### 1. Install the Wazuh server

Installed the Wazuh all-in-one stack on the Ubuntu VM. The installer finished successfully and printed the dashboard URL and admin credentials (redacted in screenshots).

### 2. Find the manager's IP and register the agent

Used `ip a` to get the VM's bridged IP address, then registered the Windows host on the manager:

```bash
ip a
sudo /var/ossec/bin/manage_agents
# (A)dd an agent -> name: WinddowsHost
# (E)xtract key for the agent
```

<img width="1905" height="967" alt="photo_6293783554906854139_w" src="https://github.com/user-attachments/assets/3eb2edbc-7068-4eba-96e4-c2793b0731c0" />

### 3. Install and enroll the Windows agent

Installed the Wazuh agent on Windows, entered the **manager IP** and the **authentication key** extracted in the previous step, and restarted the agent. The agent status showed **Running**.

<img width="396" height="345" alt="photo_6293783554906854099_x" src="https://github.com/user-attachments/assets/b7d8b068-ea2b-420e-be8e-758361443a7a" />

### 4. Verify the agent in the dashboard

The agent appeared as **Active** (Windows 11 Home, group `default`, version 4.14.9).

<img width="1917" height="971" alt="photo_6293783554906854105_w" src="https://github.com/user-attachments/assets/714275df-8dde-4e9e-b96d-6ba154a49a19" />
<img width="1917" height="972" alt="photo_6293783554906854100_w" src="https://github.com/user-attachments/assets/def00bf5-a572-4d3c-8545-ec1ac085edf7" />

### 5. Configure FIM

Edited the agent's `ossec.conf` and added a monitored directory with `realtime="yes"` so changes are reported immediately rather than waiting for the periodic scan (default every 12 hours):

```xml
<syscheck>
  <disabled>no</disabled>
  <frequency>43200</frequency>
  ...
  <directories realtime="yes">C:\Users\<user>\OneDrive\Desktop\wazuh-test</directories>
</syscheck>
```

Restarted the agent to apply the change.

<img width="1917" height="855" alt="photo_6293783554906854102_w" src="https://github.com/user-attachments/assets/406200b7-bbfb-417f-96cf-460a23699f26" />

### 6. Generate test events

In the monitored `wazuh-test` folder I created, renamed, deleted, re-created, and edited a text file (`test1.txt`).

## Results

Wazuh detected every change and mapped each to a rule:

| Action | Rule ID | Level | Description |
|---|---|---|---|
| File created | 554 | 5 | File added to the system |
| File deleted | 553 | 7 | File deleted |
| File content modified | 550 | 7 | Integrity checksum changed |

For modified files, Wazuh recorded the **old and new MD5, SHA-1, and SHA-256 hashes**, which is how FIM proves a file's content changed. Events were tagged with `Mode: realtime`, confirming real-time monitoring worked.

<img width="1917" height="961" alt="photo_6293783554906854101_w" src="https://github.com/user-attachments/assets/9bbc708b-b7c8-4809-9537-cf56c07ce9a1" />
<img width="1917" height="966" alt="photo_6293783554906854104_w" src="https://github.com/user-attachments/assets/b9f48b28-cf6d-46d1-a8e8-983367ae1b52" />
<img width="1917" height="972" alt="photo_6293783554906854103_w" src="https://github.com/user-attachments/assets/5dcece37-bb45-4f00-a2eb-cab6fb08068f" />
<img width="1917" height="966" alt="photo_6293783554906854131_w (1)" src="https://github.com/user-attachments/assets/538bddc5-bc59-4bab-8cc6-25f550fe3616" />
<img width="1917" height="966" alt="photo_6293783554906854130_w" src="https://github.com/user-attachments/assets/ff55d457-06a5-48a2-b20d-c0f418269b5b" />

The agent overview also showed Wazuh's built-in mapping of findings to **MITRE ATT&CK** (Defense Evasion) and compliance frameworks such as **PCI DSS**.

## Key Learnings

- How a Wazuh manager and agent communicate and how agents are enrolled with keys
- How FIM works: baseline, hash comparison, and alerting on add/modify/delete
- The difference between scheduled scans and `realtime` monitoring in `syscheck`
- How to read Wazuh alerts (rule ID, level, decoder, `full_log`)
- Bridged networking requirements for host-to-VM agent connectivity

## Limitations

- Single agent and a single test directory
- Test events were generated by me, not by real malware or an attacker
- No custom rules, active responses, or alert notifications yet

## Security Notes

- Change the default Wazuh admin credentials and store them securely.
- Never publish agent authentication keys, dashboard passwords, or your public IP; they are redacted in these screenshots.
- Keep the manager on a trusted network and don't expose port 443 or the agent ports (1514/1515) to the internet.
- Monitor only the directories you need; very broad FIM paths create noisy alerts.

## Next Steps

- Monitor sensitive paths (e.g., Startup folder, `System32\drivers\etc`)
- Write a custom rule to raise severity for specific files
- Add an active response or email/Slack alerting
- Add a Linux agent and compare FIM behavior across OSes
- Test detection of simulated attacker behavior (e.g., persistence in the Startup folder)
