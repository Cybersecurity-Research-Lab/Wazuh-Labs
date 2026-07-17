# Minimal Setup for Wazuh Labs

This is *not* a production deployment guide. It's the smallest setup that gets you generating and viewing real alerts so you can follow along with the labs in this repo. For a full production install, see the [official Wazuh documentation](https://documentation.wazuh.com/).

## What you need

- A Linux VM or container for the **Wazuh manager** (the server that receives and analyzes logs)
- A second Linux VM (or the same host) to act as the **monitored endpoint**, running the Wazuh agent
- Docker, if you want the fastest path (recommended for lab purposes)

## Option A: Docker (fastest)

Wazuh publishes an official single-node Docker deployment for testing:

```bash
git clone https://github.com/wazuh/wazuh-docker.git -b v4.9.0
cd wazuh-docker/single-node
docker compose up -d
```

This spins up:
- `wazuh.manager` — the core analysis engine
- `wazuh.indexer` — stores alerts (OpenSearch-based)
- `wazuh.dashboard` — web UI to view alerts

Once it's running, the dashboard is available at `https://localhost:443` (default credentials are in the Wazuh docs — change them before doing anything beyond local testing).

## Option B: Manual install

If you'd rather install directly on a VM:

```bash
curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

This installs manager, indexer, and dashboard together on one host. Credentials are printed at the end of the install — save them.

## Connecting an agent (the monitored endpoint)

On the machine you want to monitor:

```bash
# Debian/Ubuntu example
curl -o wazuh-agent.deb https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.9.0-1_amd64.deb
sudo WAZUH_MANAGER='<manager-ip>' dpkg -i ./wazuh-agent.deb
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

Replace `<manager-ip>` with the IP address of your manager. Confirm the agent shows as **Active** in the dashboard under Agents before moving on.

## Confirming alerts are flowing

Trigger a quick test alert — for example, a failed SSH login on the monitored endpoint:

```bash
ssh invaliduser@<monitored-endpoint-ip>
```

Then check the dashboard's Security Events view. You should see an alert appear within a few seconds. If you see it, you're ready for [lab-01-brute-force-ssh](../lab-01-brute-force-ssh).

## Troubleshooting

- **No alerts showing up** — check the agent is "Active" in the dashboard, and that the manager's firewall allows traffic on port 1514 (agent registration) and 1515 (agent-manager communication).
- **Dashboard won't load** — give the containers/services a minute or two after startup; the indexer can take a while to come up.

## Notes

- This setup is for learning purposes only. Do not expose these services to the public internet.
- Everything here runs fine on a single laptop with ~8GB RAM available for the VM/containers.
