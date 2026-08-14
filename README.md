# Wazuh SIEM Homelab

A self-deployed Wazuh SIEM/XDR stack, built as a hands-on detection engineering project to develop and demonstrate practical security engineering skills — deploying a real SIEM, monitoring live infrastructure, and validating detections against actual attack simulations.

## Architecture

- **Wazuh Manager, Indexer, and Dashboard** deployed via Docker Compose (official single-node deployment) on a dedicated Ubuntu Server 24.04 LTS VM, provisioned on Proxmox
- **Agent** installed on a live homelab server running 35+ Docker containers (media, productivity, monitoring, and security services), providing real, active telemetry rather than a sterile test environment
- Manager and agent communicate over the standard Wazuh ports (1514/1515)

![Dashboard Overview](docs/images/dashboard-overview.png)

## Deployment

1. Provisioned a dedicated VM (4 vCPU, 16GB RAM, 80GB disk) on Proxmox
2. Installed Docker and configured `vm.max_map_count` for the Indexer (OpenSearch) per Wazuh's requirements
3. Deployed the official `wazuh-docker` single-node stack (v4.14.7) via Docker Compose
4. Generated the required indexer SSL certificates and brought up Manager, Indexer, and Dashboard
5. Deployed a Wazuh agent to a live homelab server and confirmed active connectivity from the dashboard

### Issue encountered: Dashboard index pattern timeout on first boot

On first launch, the Dashboard reported timeout errors ("[Alerts index pattern] timeout of 20000ms exceeded") while attempting to initialize its index patterns. Root cause was a startup race condition — the Indexer (OpenSearch) takes longer to fully initialize its internal indices than the Dashboard's default timeout allows for. Resolved by allowing the Indexer additional time to complete startup, then retrying the index pattern checks from the Dashboard UI.

![Agent Connected](docs/images/agents-connected.png)

## Detection: SSH Brute-Force Simulation

To validate the deployment was generating real, correctly-classified detections (not just passively running), I simulated a credential-guessing attack against the monitored agent's SSH service.

**Initial attempt and correction:** My first test used a valid username with an intentionally wrong password, expecting an authentication failure. Instead, Wazuh logged an *authentication success*. Investigation showed SSH key-based authentication was silently succeeding ahead of the password attempt, since the client had a valid key on file for that account. Corrected the test by using a nonexistent username, which forced SSH to reject the connection at the authentication layer rather than falling back to key auth.

**Result:** Wazuh correctly detected and escalated the attack pattern:

| Rule ID | Level | Description |
|---|---|---|
| 5710 | 5 | sshd: Attempt to login using a non-existent user |
| 5503 | 5 | PAM: User login failed |
| 2502 | 10 | syslog: User missed the password more than one time (correlation rule) |

The individual failed attempts were logged at level 5, and Wazuh's correlation engine escalated to a level 10 alert after detecting the repeated-failure pattern — correctly classified under the MITRE ATT&CK techniques **Password Guessing** and **Brute Force**.

![Brute-Force Detection - Dashboard View](docs/images/brute-force-detection.png)

![Brute-Force Detection - Event Detail](docs/images/events-detail.png)

## Lessons Learned

- SIEM detection testing requires understanding the full authentication chain (key vs. password auth), not just triggering a "failed login" at face value
- Correlation rules (like 2502) provide meaningfully higher-fidelity signal than individual auth-failure events, which is why tuning and understanding rule escalation logic matters as much as deploying the tool itself
- First-boot timing issues between distributed components (Indexer vs. Dashboard) are a normal part of deploying multi-container security tooling, not a misconfiguration

## Next Steps

- [ ] Configure File Integrity Monitoring (FIM) on key infrastructure paths
- [ ] Write a custom detection rule
- [ ] Enable vulnerability detection module
- [ ] Add a second agent
