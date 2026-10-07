# Wazuh Beginner Guide

This guide is written for someone who has never used Wazuh before. It shows the simplest way to launch Wazuh locally with Docker and then test it with a sample dataset.

## 1. Prerequisites

Before you start, make sure you have:

- Docker installed and running
- Docker Compose available
- A Linux host or WSL2 if you are using Windows
- At least 4 GB of RAM available

If you are using Windows, Docker Desktop with WSL2 is the easiest option.

## 2. Clone the Wazuh Docker repository

Open a terminal and run:

```bash
git clone https://github.com/wazuh/wazuh-docker.git
cd wazuh-docker/single-node
```

## 3. Prepare the host system

On Linux, increase the memory mapping setting:

```bash
sudo sysctl -w vm.max_map_count=262144
```

This is required for the Wazuh indexer to start correctly.

## 4. Generate certificates

Run the certificate generator:

```bash
docker compose -f generate-indexer-certs.yml run --rm generator
```

If your environment uses the older syntax, try:

```bash
docker-compose -f generate-indexer-certs.yml run --rm generator
```

## 5. Start Wazuh

Start the stack in the foreground:

```bash
docker compose up
```

Or start it in the background:

```bash
docker compose up -d
```

The first startup may take 2 to 5 minutes.

## 6. Verify that Wazuh is running

After the containers are up, open your browser and visit:

- http://localhost:443

Default login credentials for this Docker example are:

- Username: admin
- Password: SecretPassword

You should now see the Wazuh dashboard.

## 7. Basic checks

You can verify the services with:

```bash
docker compose ps
```

You should see containers for the manager, indexer, and dashboard.

## 8. Deploy a sample dataset

The easiest way to test Wazuh is to send sample logs to it. You can do this by creating a simple log file and forwarding it to the Wazuh manager.

### Option A: Use a sample log file

Create a test log file:

```bash
mkdir -p /tmp/wazuh-test
cat > /tmp/wazuh-test/sample.log <<'EOF'
2026-07-21T10:00:00Z user=alice action=login result=success
2026-07-21T10:01:00Z user=bob action=failed_login result=failed
2026-07-21T10:02:00Z user=alice action=logout result=success
EOF
```

Then copy it into the manager container:

```bash
docker cp /tmp/wazuh-test/sample.log wazuh-manager:/var/ossec/logs/sample.log
```

If the container name is different in your setup, check it with:

```bash
docker ps
```

### Option B: Use a test agent

If you want a more realistic setup, install a Wazuh agent on another machine or VM, then configure it to send data to the manager.

Typical agent registration steps are:

1. Install the Wazuh agent on the target host.
2. Configure the agent to point to your Wazuh manager.
3. Start the agent service.
4. Check the Wazuh dashboard for incoming alerts.

## 9. Check alerts in the dashboard

After sending test data:

1. Open the Wazuh dashboard.
2. Go to the Security events or Alerts view.
3. Look for recent events related to your sample log.

## 10. Stop and clean up

To stop everything:

```bash
docker compose down
```

To remove volumes as well:

```bash
docker compose down -v
```

## 11. Troubleshooting tips

If the dashboard does not load:

- Wait a few more minutes for the services to initialize.
- Check container health with `docker compose ps`.
- Review logs with `docker compose logs`.

If you cannot log in:

- Confirm the credentials are `admin` and `SecretPassword`.
- Make sure the containers are fully started.

## 12. Summary

In short, the basic workflow is:

1. Install Docker and Docker Compose.
2. Start the Wazuh Docker stack.
3. Open the dashboard.
4. Send sample logs or connect an agent.
5. Review alerts in the dashboard.

This is a good beginner-friendly way to learn how Wazuh works before moving to a real production deployment.
