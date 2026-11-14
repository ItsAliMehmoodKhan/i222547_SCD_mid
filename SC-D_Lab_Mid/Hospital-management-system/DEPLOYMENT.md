# Production deployment (Ubuntu)

This deployment uses Nginx for the public entry point and static frontend, and
systemd to run and restart the API independently of terminal sessions. The
backend listens only on `127.0.0.1:4001`; clients use the server IP on port 80.

## Requirements

- Ubuntu server with a public IP address and sudo access.
- Node.js 20.19 or later installed system-wide, with the executable at
  `/usr/bin/node`.
- Nginx and npm.

## Install the application

Copy the `Hospital-management-system` directory to the server at
`/var/www/hospital-management-system`. For example, from a local machine:

```sh
scp -r Hospital-management-system user@SERVER_IP:/tmp/hospital-management-system
ssh user@SERVER_IP
sudo mkdir -p /var/www
sudo mv /tmp/hospital-management-system /var/www/hospital-management-system
```

Install backend dependencies and build the frontend:

```sh
cd /var/www/hospital-management-system/backend
sudo npm install --omit=dev

cd /var/www/hospital-management-system/frontend
sudo npm ci
sudo npm run build
```

The resulting static site is in `frontend/dist`; Node.js does not serve these
files.

## Configure automatic startup and routing

Install and enable the systemd backend service and Nginx site:

Before enabling Nginx, edit `/etc/nginx/sites-available/hospital-management`
after copying the file below and replace `SERVER_IP` in `server_name` with the
server's public IP address. If another Nginx site is already configured for
that same IP, merge these locations into that site's server block instead of
enabling a duplicate.

```sh
sudo cp /var/www/hospital-management-system/deploy/systemd/hospital-appointment-api-I222547.service /etc/systemd/system/
sudo cp /var/www/hospital-management-system/deploy/nginx/hospital-management.conf /etc/nginx/sites-available/hospital-management
# Edit server_name in /etc/nginx/sites-available/hospital-management here.
sudo ln -s /etc/nginx/sites-available/hospital-management /etc/nginx/sites-enabled/hospital-management
sudo systemctl daemon-reload
sudo systemctl enable --now hospital-appointment-api-I222547.service
sudo nginx -t
sudo systemctl enable --now nginx
sudo systemctl reload nginx
```

The systemd service is registered as `hospital-appointment-api-I222547`. It
starts at boot and restarts the backend after an unexpected exit. Nginx serves
frontend routes and forwards `/api/` and `/health` requests to the loopback-only
API. Port `4001` does not need to be opened in the firewall.

Allow inbound HTTP if UFW is enabled:

```sh
sudo ufw allow 80/tcp
```

## Verify

On the server:

```sh
sudo systemctl status hospital-appointment-api-I222547.service
curl http://127.0.0.1:4001/health
curl http://127.0.0.1/health
```

From a browser, visit `http://SERVER_IP/`; the API health endpoint is available
at `http://SERVER_IP/health`. Check backend logs with:

```sh
sudo journalctl -u hospital-appointment-api-I222547.service -f
```

After deploying frontend changes, rebuild `frontend/dist` and reload Nginx.
After backend changes, restart the service with
`sudo systemctl restart hospital-appointment-api-I222547.service`.
