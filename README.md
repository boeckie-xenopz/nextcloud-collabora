# Nextcloud with Collabora and Traefik

This repository contains a Docker Compose configuration that deploys a self-hosted Nextcloud instance along with its supporting services. Nextcloud is an open-source cloud storage and collaboration platform that lets you host your own file-sharing and productivity environment.  
In this setup, Nextcloud is paired with:

- **Postgres:** A robust relational database used by Nextcloud for storing its data.  
- **Redis:** An in-memory key-value store used by Nextcloud for caching, file-locking and overall performance improvements.  
- **Collabora Online:** An integrated office suite that allows collaborative document editing.  
- **notify_push:** A real-time push-notification server that makes desktop & mobile clients react instantly to file changes.  
- **Traefik:** A modern reverse proxy that automatically routes incoming HTTP/HTTPS requests to the correct container and handles SSL termination using Let’s Encrypt.

---

## Repository Structure and Service Overview

### Docker Compose File

The core of this deployment is the `docker-compose.yaml` file, which defines the following services:

- **Traefik:**  
  - Acts as the reverse proxy.  
  - Listens on ports 80 (HTTP), 443 (HTTPS), and 8080 (dashboard).  
  - Automatically manages SSL certificates via the ACME protocol.  
  - Dynamically routes requests to Nextcloud, Collabora **and notify_push** based on host rules defined in container labels.  
  - Uses Docker labels to secure and customize routes (e.g., enabling HTTP-to-HTTPS redirection and setting security headers).

- **Postgres:**  
  - Provides a persistent relational database for Nextcloud.  
  - Is configured through environment variables (database name, user, and password).  
  - Uses a dedicated volume to persist data.

- **Redis:**  
  - Supplies in-memory caching and file-locking for Nextcloud, greatly improving responsiveness and scalability.  
  - Runs with append-only persistence enabled (`--appendonly yes`) so cache data can survive container restarts.  
  - Exposed only to the internal Docker network; Nextcloud connects via the `REDIS_HOST` and `REDIS_HOST_PORT` environment variables.

- **Nextcloud:**  
  - The main application offering file sharing, syncing, and collaboration.  
  - Connects to the Postgres service to store its data and to Redis for caching/file-locking.  
  - Exposes WebDAV, Cal/CardDAV and the standard web UI through Traefik.  
  - Uses additional middleware (e.g., for proper handling of WebDAV and secure headers).

- **Collabora Online:**  
  - Enables online document editing, integrated with Nextcloud.  
  - Is accessible through Traefik with its own host rule (`office.your_domain.tld`).  
  - Uses environment variables to integrate with Nextcloud and configure language dictionaries.  
  - Runs on a specific port (9980) that Traefik routes to via container labels.

- **notify_push:**  
  - Provides real-time push notifications for Nextcloud clients (desktop & mobile) and the web interface.  
  - Listens on port `7867` inside the container; Traefik routes external requests hitting `https://cloud.your_domain.tld/push` to this port.  
  - Reads the Nextcloud installation directory in **read-only** mode (`/var/www/html:ro`) to access the generated authentication keys.  
  - Connects to Redis for caching and to Postgres to fetch required data.  
  - Speaks to Nextcloud via internal HTTP calls, enabling instant propagation of file-change events without polling.

---

## System Architecture Diagram

```mermaid
graph TD
    A[External User]
    B[Traefik Reverse Proxy]
    C[Nextcloud Service]
    D[Postgres Database]
    E[Collabora Online]
    F[Redis Cache]
    G[notify_push]

    A -->|HTTP/HTTPS Requests| B
    B -->|Routes cloud.your_domain.tld<br/>(web, WebDAV)| C
    B -->|Routes office.your_domain.tld| E
    B -->|Routes /push on cloud.your_domain.tld| G
    C -->|Database Connections| D
    C -->|Caching / File Locking| F
    C -->|Local HTTP| G
    E <-->|WOPI Protocol| C
```

*Figure – Traefik routes incoming requests to Nextcloud, Collabora and notify_push. Nextcloud uses Postgres for data storage, Redis for caching/file-locking, and notify_push for real-time client notifications.*

---

## How Traefik Works in This Setup

Traefik is a dynamic reverse proxy designed to work seamlessly with Docker. Here’s how it functions in this deployment:

- **Dynamic Configuration:**  
  Traefik monitors Docker containers for specific labels. In this configuration, Nextcloud, Collabora and notify_push containers include Traefik labels that define:  
  - **Routing Rules:** Which hostnames and paths should be routed to each service.  
  - **TLS Settings:** Enabling HTTPS and specifying certificate resolvers to automatically request SSL certificates from Let’s Encrypt.  
  - **Middleware:** Additional settings such as redirections, custom headers, and security policies.

- **SSL Termination:**  
  Traefik listens on port 80 and redirects traffic to HTTPS on port 443. It uses the ACME protocol (configured via command-line arguments) to obtain and renew SSL certificates automatically.

- **Dashboard and Logging:**  
  Traefik provides an administrative dashboard on port 8080. Access to the dashboard is secured with basic authentication. Logging and access logs are configured to help in monitoring and troubleshooting.

- **Integration via Docker Socket:**  
  By mounting the Docker socket, Traefik detects and adapts to changes in container state automatically, without requiring manual configuration changes.

In summary, Traefik simplifies routing and security for the Nextcloud stack by automating SSL certificate management and dynamically adapting to container changes—all defined directly in the Docker Compose file.

---

## Customization and Usage

- **Service Modifications:**  
  You can extend or modify the services as needed (e.g., updating images, adding new environment variables, or integrating additional middleware such as Fail2Ban).

- **Persistent Storage:**  
  Make sure the volumes defined for Traefik (`/data/traefik`), Postgres (`/data/postgres`), Redis (`/data/redis`) **and Nextcloud (`/data/nextcloud`)** are configured for persistent storage on your host.  
  The **notify_push** service mounts the same Nextcloud volume **in read-only mode**, so no additional volume is required for it.

- **Environment Settings:**  
  Adjust time zones, credentials, and trusted domains in the environment variables for each service to match your setup.

- **Security Considerations:**  
  The Traefik dashboard is protected via basic authentication. Ensure that you update these credentials and review other security settings to maintain a secure deployment.

---

## Nextcloud configuration

If you used the original Docker Compose configuration, you will find your **Nextcloud** install at `https://cloud.your_domain.tld/`.  
If necessary, first create an admin account and password.

Next, install **Nextcloud Office** (Collabora integration).  
Then go to **Administration Settings** → **Nextcloud Office** and configure the Collabora Online URL (`https://office.your_domain.tld`).

---

## Post-Installation Steps (notify_push)

After all containers are up and running you must register the push-notification endpoint inside your Nextcloud instance.

1. **Run the OCC setup command**

   ```bash
   # run as root on the Docker host
   docker exec -u www-data nextcloud php occ notify_push:setup https://cloud.your_domain.tld/push
   ```

2. **What this command does**

   - Generates a cryptographic authentication token used by **notify_push**  
   - Creates the required entries in the Nextcloud database  
   - Stores the push endpoint (`/push`) so that clients can establish a persistent websocket connection  
   - Without this step, clients will fallback to periodic polling, negating the performance benefits of real-time notifications.

3. **Verify that notify_push is working**

   - Check container logs: `docker logs -f notify_push` – you should see a line like `Server started on 0.0.0.0:7867`.  
   - Inside the Nextcloud container run:  
     ```bash
     docker exec -u www-data nextcloud php occ notify_push:self-test
     ```  
     The test should return `✓ success`.
   - From a client machine:  
     `curl -I https://cloud.your_domain.tld/push` should show `HTTP/1.1 101 Switching Protocols`.

4. **Troubleshooting tips**

   - **404 or 502 errors** on `/push`: ensure Traefik labels for the `notify_push` service match your cloud domain and path.  
   - **Self-test fails**: verify Redis connectivity (`docker exec nextcloud php occ config:system:get redis`) and Postgres credentials.  
   - **Websocket closed immediately**: double-check that the `notify_push` container is running and no firewall or proxy blocks port 7867 internally.  
   - If you changed the domain later, rerun the `notify_push:setup` command with the new URL.

---

## Conclusion

This repository provides an out-of-the-box solution for deploying Nextcloud with an integrated office suite (Collabora), real-time push notifications (notify_push) and a robust reverse proxy (Traefik) that manages secure connections and dynamic routing.  
The inclusion of Redis further enhances performance through caching and reliable file-locking.  
The Docker Compose file is designed to streamline the setup process while ensuring that each component is properly isolated, secure, and easily extendable.

## TODO

- Write an env file containing all configuration settings, in particular the Nextcloud and Collabora domains  
- Make the install more robust with  
  - an external and properly managed PostgreSQL instance  
  - several container instances  

## References

- [Docker Compose for Nextcloud + Collabora + Traefik?](https://help.nextcloud.com/t/docker-compose-for-nextcloud-collabora-traefik/127733/6)  
- [Docker compose - Collabora - Traefik](https://help.nextcloud.com/t/docker-compose-collabora-traefik/219975)  
- [How to docker-compose with notify_push (2024)](https://help.nextcloud.com/t/how-to-docker-compose-with-notify_push-2024/186721)  
