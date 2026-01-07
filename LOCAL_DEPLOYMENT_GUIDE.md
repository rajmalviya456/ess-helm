# ESS Community - Local Development Guide

Complete guide to running Element Server Suite Community locally for development and testing.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Initial Setup](#initial-setup)
3. [Configuration Explained](#configuration-explained)
4. [Deployment](#deployment)
5. [User Management](#user-management)
6. [Accessing Services](#accessing-services)
7. [How It Works](#how-it-works)
8. [Data Storage](#data-storage)
9. [Cleanup](#cleanup)
10. [Troubleshooting](#troubleshooting)

---

## Prerequisites

Install the following tools on macOS:

```bash
# Install Homebrew (if not already installed)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install k3d (Kubernetes in Docker)
brew install k3d

# Install Helm
brew install helm

# Install kubectl (if not already installed)
brew install kubectl
```

**System Requirements:**
- macOS (any recent version)
- Docker Desktop running
- At least 2 CPU cores and 4GB RAM available

---

## Initial Setup

### 1. Clone the Repository

```bash
git clone https://github.com/element-hq/ess-helm.git
cd ess-helm
```

### 2. Create the Test Cluster

```bash
# This creates a local k3d cluster with ingress, cert-manager, and self-signed CA
./scripts/setup_test_cluster.sh
```

**What this does:**
- Creates a k3d cluster named `ess-helm`
- Installs Traefik ingress controller (bound to localhost:80 and localhost:443)
- Installs cert-manager for TLS certificates
- Creates a self-signed CA in `./.ca/` directory
- Sets up the `ess` namespace

### 3. Trust the Self-Signed CA Certificate

```bash
# Add CA to macOS keychain (requires password)
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain .ca/ca.crt
```

**Important:** Restart your browser completely after this step.

### 4. Deploy MailHog (Email Testing Server)

```bash
# Deploy MailHog for capturing emails
kubectl -n ess create deployment mailhog --image=mailhog/mailhog:latest -- MailHog

# Expose MailHog services
kubectl -n ess expose deployment mailhog --port=8025 --target-port=8025 --name=mailhog-web
kubectl -n ess expose deployment mailhog --port=1025 --target-port=1025 --name=mailhog-smtp

# Create ingress for web UI
cat <<EOF | kubectl -n ess apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mailhog-web-alt
  annotations:
    cert-manager.io/cluster-issuer: ess-selfsigned
spec:
  ingressClassName: traefik
  tls:
  - hosts:
    - mail.ess.localhost
    secretName: mailhog-web-tls
  rules:
  - host: mail.ess.localhost
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mailhog-web
            port:
              number: 8025
EOF
```

### 5. Create Configuration File

Create the file `charts/matrix-stack/user_values/test-local.yaml`:

```yaml
# Local test cluster configuration
# This allows insecure HTTP URIs for OAuth client registration

elementWeb:
  additional:
    ui-features.json: |
      {
        "setting_defaults": {
          "UIFeature.passwordReset": true
        }
      }

matrixAuthenticationService:
  additional:
    test-local-config.yaml:
      config: |
        # Email configuration for local testing with MailHog
        email:
          from: '"ESS Local" <noreply@ess.localhost>'
          reply_to: '"ESS Support" <support@ess.localhost>'
          transport: smtp
          hostname: mailhog-smtp.ess.svc.cluster.local
          port: 1025
          mode: plain

        # Account settings - enable password recovery
        account:
          password_change_allowed: true
          password_recovery_enabled: true

        # Password policy - allow weak passwords for local testing
        passwords:
          enabled: true
          minimum_length: 4
          require_uppercase: false
          require_lowercase: false
          require_number: false
          require_symbol: false

        # Policy to allow insecure URIs for local development
        policy:
          data:
            client_registration:
              allow_insecure_uris: true
              allow_host_mismatch: true
```

---

## Configuration Explained

### Element Web Configuration

```yaml
elementWeb:
  additional:
    ui-features.json: |
      {
        "setting_defaults": {
          "UIFeature.passwordReset": true  # Enables "Forgot Password" button
        }
      }
```

**What it does:** Enables the password reset UI feature in Element Web client.

### Matrix Authentication Service (MAS) Configuration

#### 1. Email Settings

```yaml
email:
  from: '"ESS Local" <noreply@ess.localhost>'
  reply_to: '"ESS Support" <support@ess.localhost>'
  transport: smtp
  hostname: mailhog-smtp.ess.svc.cluster.local  # Internal k8s service name
  port: 1025
  mode: plain
```

**What it does:**
- Configures MAS to send emails via MailHog SMTP server
- Uses plain mode (no encryption) - OK for local testing
- Internal hostname resolves within the Kubernetes cluster

#### 2. Account Settings

```yaml
account:
  password_change_allowed: true      # Users can change their password
  password_recovery_enabled: true    # Enables "Forgot Password" flow
```

**What it does:**
- Allows users to change their password from account settings
- Enables password recovery via email

#### 3. Password Policy

```yaml
passwords:
  enabled: true
  minimum_length: 4              # Only 4 characters required
  require_uppercase: false       # No uppercase required
  require_lowercase: false       # No lowercase required
  require_number: false          # No numbers required
  require_symbol: false          # No special characters required
```

**What it does:**
- Relaxes password requirements for easy local testing
- ⚠️ **DO NOT USE IN PRODUCTION** - This is insecure!

#### 4. OAuth Client Policy

```yaml
policy:
  data:
    client_registration:
      allow_insecure_uris: true     # Allows HTTP redirect URIs
      allow_host_mismatch: true     # Allows different hostnames
```

**What it does:**
- Allows Element Web/Admin to dynamically register with HTTP URLs
- Required for local development with self-signed certificates
- ⚠️ **DO NOT USE IN PRODUCTION**

---

## Deployment

### Deploy the Matrix Stack

```bash
helm -n ess upgrade -i ess charts/matrix-stack \
  -f charts/matrix-stack/ci/test-cluster-mixin.yaml \
  -f charts/matrix-stack/ci/example-default-enabled-components-values.yaml \
  -f charts/matrix-stack/user_values/test-local.yaml
```

**What this deploys:**
- Synapse (Matrix homeserver)
- Matrix Authentication Service (User management & OAuth)
- Element Web (Web chat client)
- Element Admin (Admin console)
- Matrix RTC (Real-time communication for calls)
- PostgreSQL (Database)
- HAProxy (Load balancer)
- Well-known delegation service

### Wait for Deployment

```bash
# Check pod status
kubectl -n ess get pods

# Wait for all pods to be ready
kubectl -n ess wait --for=condition=ready pod --all --timeout=5m
```

---

## User Management

### Create the First User

```bash
kubectl exec -n ess -it deployment/ess-matrix-authentication-service -- mas-cli manage register-user
```

**Interactive prompts:**
1. Username: `testuser`
2. Email: `testuser@example.com`
3. Display name: `Test User`
4. Make admin: `yes`
5. Password: `test1234` (min 4 chars)

### Add Email to Existing User

```bash
kubectl exec -n ess deployment/ess-matrix-authentication-service -- \
  mas-cli manage add-email <username> <email@example.com>
```

### Set/Reset User Password

```bash
kubectl exec -n ess deployment/ess-matrix-authentication-service -- \
  mas-cli manage set-password <username>
```

---

## Accessing Services

### Service URLs

| Service | URL | Purpose |
|---------|-----|---------|
| Element Web | https://element.ess.localhost | Main chat client |
| Element Admin | https://admin.ess.localhost | Admin console |
| MAS Account | https://mas.ess.localhost | Account management |
| Synapse | https://synapse.ess.localhost | Matrix homeserver API |
| Matrix RTC | https://mrtc.ess.localhost | Real-time communication |
| MailHog | https://mail.ess.localhost | Email testing UI |
| Well-Known | https://ess.localhost/.well-known/matrix/client | Matrix discovery |

### Login to Element Web

1. Go to https://element.ess.localhost
2. Click **"Sign in"**
3. Server name: `ess.localhost`
4. Username: `testuser`
5. Password: `test1234`

### Password Reset Testing

1. Go to https://element.ess.localhost
2. Click **"Sign in"** → **"Forgot password?"**
3. Enter your email: `testuser@example.com`
4. Check https://mail.ess.localhost for reset email
5. Click the link in the email to reset password

---

## How It Works

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     Browser (https://*.ess.localhost)        │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│              Traefik Ingress (localhost:443)                 │
│              - TLS termination with self-signed certs        │
└──────────────────┬──────────────────────────────────────────┘
                   │
      ┌────────────┼────────────┐
      ▼            ▼            ▼
┌──────────┐ ┌──────────┐ ┌─────────┐
│ Element  │ │   MAS    │ │ Synapse │
│   Web    │ │  (Auth)  │ │(Matrix) │
└──────────┘ └────┬─────┘ └────┬────┘
                  │            │
                  ▼            ▼
            ┌──────────────────────┐
            │    PostgreSQL        │
            │  - MAS database      │
            │  - Synapse database  │
            └──────────────────────┘
```

### Chat & Messaging

**How Messages Work:**

1. **Sending a message:**
   - User types message in Element Web
   - Element Web sends to Synapse via `/sync` and `/send` APIs
   - Synapse stores in PostgreSQL database
   - Synapse forwards to other participants and federated servers

2. **Receiving messages:**
   - Element Web polls Synapse `/sync` endpoint
   - Synapse returns new messages from database
   - Element Web displays messages in UI

3. **Message Storage:**
   - Database: PostgreSQL (`ess-postgres` pod)
   - Location: `/var/lib/postgresql/data` (PVC: `ess-postgres-data`)
   - Encryption: Messages encrypted end-to-end (keys stored client-side)

### Voice & Video Calls

**How Calls Work:**

1. **Call Initiation:**
   - User clicks call button in Element Web
   - Element Web requests call setup from MAS
   - MAS creates LiveKit room via Matrix RTC service

2. **Media Flow:**
   - WebRTC connection established between clients
   - Media streams through Matrix RTC SFU (Selective Forwarding Unit)
   - Real-time audio/video packets flow peer-to-peer or via SFU

3. **Components:**
   - **Matrix RTC SFU** (`ess-matrix-rtc-sfu`): LiveKit-based media server
   - **Matrix RTC Auth** (`ess-matrix-rtc-authorisation-service`): Authorizes call access
   - **Element Call**: Built into Element Web for calling UI

4. **Call Types:**
   - **1:1 calls**: Direct peer-to-peer (with STUN/TURN if needed)
   - **Group calls**: Routed through SFU for better performance
   - **Video rooms**: Persistent call spaces in rooms

### Media Storage

**Where Media Files Are Stored:**

```
┌─────────────────────────────────────────────────────────┐
│              Media Upload Flow                          │
└─────────────────────────────────────────────────────────┘

User uploads file (image, video, etc.)
         │
         ▼
    Element Web
         │
         ▼
    Synapse API (/upload endpoint)
         │
         ▼
┌────────────────────────┐
│  Synapse Media Store   │
│  PVC: ess-synapse-media│
│  Path: /media          │
│  Size: 10GB            │
└────────────────────────┘
         │
         ▼
   File stored with hash-based name
   (e.g., /media/local_content/...)
```

**Media File Locations:**

```bash
# Check media storage
kubectl -n ess exec ess-synapse-main-0 -- ls -lah /media/

# Media files are stored as:
# /media/local_content/<server>/<hash>/<filename>
# /media/remote_content/<server>/<hash>/<filename>
```

**Storage Volumes:**

```bash
# View persistent volumes
kubectl -n ess get pvc

# NAME                STATUS   CAPACITY
# ess-postgres-data   Bound    10Gi      # Database data
# ess-synapse-media   Bound    10Gi      # Media files
```

### Database Schema

**PostgreSQL Databases:**

1. **synapse** - Synapse homeserver data
   - Tables: users, rooms, events, state, devices, etc.
   - Messages stored as "events" in `events` table

2. **matrixauthenticationservice** - MAS data
   - Tables: users, sessions, oauth_clients, etc.
   - User accounts, OAuth tokens, session data

```bash
# Connect to PostgreSQL
kubectl -n ess exec -it ess-postgres-0 -- psql -U postgres

# List databases
\l

# Connect to synapse database
\c synapse

# List tables
\dt

# View room events (messages)
SELECT event_id, room_id, type, sender FROM events LIMIT 10;
```

---

## Data Storage

### Persistent Volumes

```bash
# View all persistent volumes
kubectl -n ess get pvc

# Backup PostgreSQL data
kubectl -n ess exec ess-postgres-0 -- pg_dump -U postgres synapse > synapse-backup.sql

# Backup media files
kubectl -n ess exec ess-synapse-main-0 -- tar czf - /media > media-backup.tar.gz
```

### Data Locations (Inside Pods)

| Component | Data Type | Path | Storage |
|-----------|-----------|------|---------|
| PostgreSQL | Databases | `/var/lib/postgresql/data` | PVC (10GB) |
| Synapse | Media files | `/media` | PVC (10GB) |
| Synapse | Config | `/conf` | ConfigMap |
| MAS | Config | `/config` | ConfigMap + Secret |

---

## Cleanup

### Stop the Cluster

```bash
# Delete the cluster (keeps volumes)
./scripts/destroy_test_cluster.sh
```

### Complete Cleanup

```bash
# Delete everything including volumes
./scripts/destroy_test_cluster.sh

# Remove persistent volumes manually if needed
kubectl delete pvc ess-postgres-data ess-synapse-media -n ess

# Remove the cluster
k3d cluster delete ess-helm
```

### Remove CA Certificate

```bash
# Remove CA from macOS keychain
sudo security delete-certificate -c "ess-ca" /Library/Keychains/System.keychain
```

---

## Troubleshooting

### Check Pod Status

```bash
# View all pods
kubectl -n ess get pods

# View pod logs
kubectl -n ess logs -f deployment/ess-synapse-main
kubectl -n ess logs -f deployment/ess-matrix-authentication-service

# Describe pod for events
kubectl -n ess describe pod <pod-name>
```

### Common Issues

#### 1. SSL Certificate Errors

**Problem:** Browser shows "Your connection is not private"

**Solution:**
```bash
# Trust the CA certificate
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain .ca/ca.crt

# Restart browser completely (Cmd+Q)
```

#### 2. Pods Not Starting

**Problem:** Pods stuck in `CrashLoopBackOff` or `Pending`

**Solution:**
```bash
# Check pod logs
kubectl -n ess logs <pod-name>

# Check events
kubectl -n ess get events --sort-by='.lastTimestamp'

# Restart deployment
kubectl -n ess rollout restart deployment/<deployment-name>
```

#### 3. OAuth Redirect Errors

**Problem:** "invalid_redirect_uri" error

**Solution:** Already fixed in `test-local.yaml` with:
```yaml
policy:
  data:
    client_registration:
      allow_insecure_uris: true
      allow_host_mismatch: true
```

#### 4. Password Reset Not Showing

**Problem:** No "Forgot password?" button

**Solutions:**
1. Hard refresh browser (Cmd+Shift+R)
2. Use incognito mode
3. Go directly to: https://mas.ess.localhost/account/password/forgot
4. Verify user has email: `kubectl exec -n ess deployment/ess-matrix-authentication-service -- mas-cli manage add-email <username> <email>`

#### 5. Emails Not Received

**Problem:** Password reset emails not appearing

**Solution:**
```bash
# Check MailHog is running
kubectl -n ess get pods | grep mailhog

# Check MAS can connect to MailHog
kubectl -n ess logs deployment/ess-matrix-authentication-service | grep -i smtp

# Access MailHog UI
open https://mail.ess.localhost
```

### Useful Commands

```bash
# Watch pod status in real-time
kubectl -n ess get pods -w

# Get all resources
kubectl -n ess get all

# Get ingress URLs
kubectl -n ess get ingress

# Restart all deployments
kubectl -n ess rollout restart deployment

# Scale down/up a deployment
kubectl -n ess scale deployment/ess-synapse-main --replicas=0
kubectl -n ess scale deployment/ess-synapse-main --replicas=1

# Get PostgreSQL password
kubectl -n ess get secret ess-generated -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d

# Execute SQL query
kubectl -n ess exec -it ess-postgres-0 -- psql -U postgres -d synapse -c "SELECT count(*) FROM events;"
```

---

## Advanced Configuration

### Enable Registration (Not Recommended)

If you want to allow public registration (not recommended for local dev):

```yaml
matrixAuthenticationService:
  additional:
    test-local-config.yaml:
      config: |
        account:
          password_registration_enabled: true
          password_registration_email_required: true
```

### Configure Real SMTP

For testing with real emails:

```yaml
email:
  transport: smtp
  hostname: smtp.gmail.com
  port: 587
  mode: starttls
  username: your-email@gmail.com
  password: your-app-password
```

### Increase Storage

```yaml
# In values file
postgres:
  persistence:
    size: 20Gi

synapse:
  persistence:
    media:
      size: 50Gi
```

---

## Next Steps

1. **Explore Element Web**: Create rooms, invite users, send messages
2. **Test Voice/Video Calls**: Start a call in a room
3. **Admin Console**: Use Element Admin to manage users and rooms
4. **Federation**: Configure federation to connect with matrix.org
5. **Monitoring**: Set up Prometheus/Grafana for metrics
6. **Production Setup**: Review [Production Deployment Guide](./docs/advanced.md)

---

## Resources

- [ESS Community Repo](https://github.com/element-hq/ess-helm)
- [Matrix Documentation](https://matrix.org/docs/)
- [Synapse Documentation](https://element-hq.github.io/synapse/latest/)
- [MAS Documentation](https://element-hq.github.io/matrix-authentication-service/)
- [Element Web Config](https://github.com/element-hq/element-web/blob/develop/docs/config.md)
- [LiveKit Documentation](https://docs.livekit.io/)

---

**Last Updated:** January 2, 2026
