# Secure LiteLLM Access via Cloudflare Access + GitHub Actions

A hardened proof-of-concept for accessing an on-premises LiteLLM instance securely from GitHub Actions runners using Cloudflare Access service tokens and Cloudflare Tunnel.

## Overview

This workflow demonstrates how to:
- Expose an on-premises LiteLLM instance through Cloudflare Tunnel (no public IP required)
- Authenticate GitHub Actions runners using Cloudflare Access service tokens
- Harden the CI/CD pipeline with network egress monitoring (StepSecurity)
- Keep all infrastructure details (host, credentials) secret from public logs

**Architecture:**
```
GitHub Actions Runner → Cloudflare Access (service token auth) → Cloudflare Tunnel → Private LiteLLM Instance
```

## Prerequisites

### 1. Cloudflare Account
- Active Cloudflare account with a domain configured
- Cloudflare Zero Trust (formerly Teams) enabled (free tier available)
- A domain or subdomain for your LiteLLM endpoint (e.g., `litellm.yourdomain.com`)

### 2. LiteLLM Server
- LiteLLM running on your local machine or on-premises
- Accessible via `localhost:<port>` (e.g., `localhost:4000`)
- No public IP or inbound firewall rules required

### 3. Cloudflare Tunnel
- `cloudflared` daemon installed on the machine running LiteLLM
- Download from [Cloudflare's install guide](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/)

### 4. GitHub Secrets
Add the following secrets to your repository (Settings → Secrets and variables → Actions):
- `LITELLM_HOST` – Your Cloudflare-protected domain (e.g., `litellm.yourdomain.com`)
- `CF_ACCESS_CLIENT_ID` – Service token client ID from Cloudflare
- `CF_ACCESS_CLIENT_SECRET` – Service token client secret from Cloudflare

> **Security tip:** Service tokens are long-lived credentials. Plan to rotate them regularly and consider implementing automated rotation via Bitwarden Secrets Manager or similar tools. [web:74][web:149]

## Setup Steps

### Step 1: Install and Configure Cloudflare Tunnel

Install `cloudflared` on the machine running LiteLLM:

```
# Example for Linux
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb
```

Authenticate with Cloudflare:

```
cloudflared tunnel login
```

Create a tunnel:

```
cloudflared tunnel create litellm-tunnel
```

This generates a tunnel ID and credentials file (save the tunnel ID).

Configure the tunnel by creating `~/.cloudflared/config.yml`:

```
tunnel: <your-tunnel-id>
credentials-file: /home/user/.cloudflared/<tunnel-id>.json

ingress:
  - hostname: litellm.yourdomain.com
    service: http://localhost:4000  # Replace with your LiteLLM port
  - service: http_status:404
```

Route DNS to your tunnel:

```
cloudflared tunnel route dns litellm-tunnel litellm.yourdomain.com
```

Start the tunnel as a service:

```
cloudflared service install
sudo systemctl start cloudflared
sudo systemctl enable cloudflared
```

Verify it's running:

```
sudo systemctl status cloudflared
```

### Step 2: Create Cloudflare Access Application

Go to [Cloudflare Zero Trust dashboard](https://one.dash.cloudflare.com/):

1. Navigate to **Access → Applications**
2. Click **Add an application** → **Self-hosted**
3. Configure the application:
   - **Application name:** LiteLLM
   - **Session duration:** 24 hours (or your preference)
   - **Application domain:** `litellm.yourdomain.com`
4. Click **Next**

5. Create an Access Policy:
   - **Policy name:** Service Token Policy
   - **Action:** Allow
   - **Include:** Select **Service Auth**
   - Click **Next** → **Add application**

### Step 3: Create a Service Token

In the Cloudflare Zero Trust dashboard:

1. Go to **Access → Service Tokens**
2. Click **Create Service Token**
3. Enter a name: `github-actions-litellm`
4. Set duration (or leave as "non-expiring" for testing)
5. Click **Generate token**
6. **IMPORTANT:** Copy both the **Client ID** and **Client Secret** immediately—the secret won't be shown again [web:74]

### Step 4: Add Service Token to Access Policy

Return to your LiteLLM application in **Access → Applications**:

1. Click your LiteLLM application
2. Go to the **Service Auth** policy you created
3. Click **Configure**
4. Under **Include**, select **Service Token** → choose `github-actions-litellm`
5. Click **Save**

### Step 5: Add GitHub Secrets

In your GitHub repository:

1. Go to **Settings → Secrets and variables → Actions**
2. Add these secrets:
   - `LITELLM_HOST` → `litellm.yourdomain.com` (no `https://`)
   - `CF_ACCESS_CLIENT_ID` → Client ID from Step 3
   - `CF_ACCESS_CLIENT_SECRET` → Client Secret from Step 3

### Step 6: Add the Workflow

Copy `.github/workflows/test-litellm-cloudflare.yml` (see below) to your repository.

## Workflow Details

The workflow uses three security layers:

### Layer 1: StepSecurity Harden-Runner
- **Mode:** `egress-policy: audit` (monitors all outbound traffic)
- **Purpose:** Detects unexpected network connections and provides visibility [web:130][web:128]
- Review logged connections in the StepSecurity dashboard after runs

### Layer 2: Cloudflare Access Service Token
- Authenticates via HTTP headers: `CF-Access-Client-Id` and `CF-Access-Client-Secret` [web:74]
- Cloudflare validates the token before allowing requests to reach your LiteLLM instance
- No VPN or network join required—just HTTP header authentication
- Tokens can be scoped to specific applications and revoked centrally

### Layer 3: Secret Management
- All credentials stored as GitHub secrets and masked in logs [web:153]
- Domain, Client ID, and Client Secret never appear in public workflow logs
- Cloudflare logs all Access events for audit trails

## Workflow File

```
name: Test LiteLLM via Cloudflare Access (hardened)

on:
  workflow_dispatch:  # Manual trigger for testing

jobs:
  test-litellm-access:
    runs-on: ubuntu-latest

    steps:
    - name: Harden the runner (egress audit only)
      uses: step-security/harden-runner@c6295a65d1254861815972266d5933fd6e532bdf # v2.11.1
      with:
        egress-policy: audit

    - name: Test LiteLLM health endpoint
      env:
        LITELLM_HOST: ${{ secrets.LITELLM_HOST }}
        CF_ACCESS_CLIENT_ID: ${{ secrets.CF_ACCESS_CLIENT_ID }}
        CF_ACCESS_CLIENT_SECRET: ${{ secrets.CF_ACCESS_CLIENT_SECRET }}
      run: |
        set -e
        echo "🧪 Testing LiteLLM health via Cloudflare Access..."
        RESPONSE="$(curl -ksS -w '\n%{http_code}' \
          -H "CF-Access-Client-Id: $CF_ACCESS_CLIENT_ID" \
          -H "CF-Access-Client-Secret: $CF_ACCESS_CLIENT_SECRET" \
          https://$LITELLM_HOST/health || true)"
        HTTP_CODE="$(echo "$RESPONSE" | tail -n1)"
        BODY="$(echo "$RESPONSE" | head -n-1)"

        echo "HTTP status: $HTTP_CODE"
        echo "Body: $BODY"

        if [ "$HTTP_CODE" != "200" ]; then
          echo "❌ Health check failed"
          exit 1
        fi
        echo "✅ Health check OK"

    - name: Test simple LiteLLM chat completion
      env:
        LITELLM_HOST: ${{ secrets.LITELLM_HOST }}
        CF_ACCESS_CLIENT_ID: ${{ secrets.CF_ACCESS_CLIENT_ID }}
        CF_ACCESS_CLIENT_SECRET: ${{ secrets.CF_ACCESS_CLIENT_SECRET }}
      run: |
        set -e
        echo "🚀 Testing LiteLLM /v1/chat/completions via Cloudflare Access..."
        curl -ksS -X POST "https://$LITELLM_HOST/v1/chat/completions" \
          -H "Content-Type: application/json" \
          -H "CF-Access-Client-Id: $CF_ACCESS_CLIENT_ID" \
          -H "CF-Access-Client-Secret: $CF_ACCESS_CLIENT_SECRET" \
          -d '{
            "model": "gpt-4",
            "messages": [
              {"role": "user", "content": "Say hello in one short sentence."}
            ],
            "max_tokens": 16
          }' | tee response.json

        if grep -q '"choices"' response.json; then
          echo "✅ LiteLLM responded successfully"
        else
          echo "❌ LiteLLM response did not contain choices"
          cat response.json
          exit 1
        fi

    - name: Summary
      if: always()
      run: |
        echo "## 🔐 Hardened LiteLLM Connectivity Test (Cloudflare)" >> $GITHUB_STEP_SUMMARY
        echo "- StepSecurity Harden-Runner: enabled (egress audit)" >> $GITHUB_STEP_SUMMARY
        echo "- Cloudflare Access service token: see logs above" >> $GITHUB_STEP_SUMMARY
        echo "- LiteLLM health + simple chat: see logs above" >> $GITHUB_STEP_SUMMARY
```

## Running the Workflow

1. Go to your repository **Actions** tab
2. Select **"Test LiteLLM via Cloudflare Access (hardened)"**
3. Click **"Run workflow"** → **"Run workflow"**
4. Monitor the logs in real time

**Expected output:**
- ✅ Harden-Runner initialized with audit egress policy
- ✅ LiteLLM health endpoint responds (HTTP 200)
- ✅ LiteLLM API responds with chat completion (contains `"choices"`)

## Troubleshooting

| Issue | Solution |
| --- | --- |
| **"403 Forbidden"** | Service token not authorized in Access policy; verify token is added to the Service Auth policy for your application [web:74] |
| **"Invalid service token"** | Token may be expired, revoked, or incorrect; regenerate token in Zero Trust dashboard and update GitHub secrets |
| **"Could not resolve host"** | DNS not configured correctly; verify `cloudflared tunnel route dns` succeeded and DNS propagated (check with `dig litellm.yourdomain.com`) |
| **"Connection refused"** | `cloudflared` service not running; check `sudo systemctl status cloudflared` and ensure tunnel config points to correct localhost port |
| **"502 Bad Gateway"** | LiteLLM not running or listening on wrong port; verify LiteLLM is running on the port specified in tunnel config |
| **"404 Not Found" on /health** | Confirm your LiteLLM health endpoint path; adjust curl URL if needed |

### Debugging Cloudflare Access

Check Access logs in the Cloudflare dashboard:
1. Go to **Zero Trust → Logs → Access**
2. Filter by your application name
3. Look for denied requests and the reason (invalid token, policy mismatch, etc.)

### Debugging Cloudflare Tunnel

Check tunnel logs:
```
sudo journalctl -u cloudflared -f
```

Verify tunnel status:
```
cloudflared tunnel info litellm-tunnel
```

## Comparing with Tailscale Approach

| Feature | Cloudflare Access | Tailscale |
| --- | --- | --- |
| **Setup complexity** | Moderate (tunnel + Access + service token) | Moderate (serve + ACLs + OAuth client) |
| **Network model** | HTTP proxy through Cloudflare edge | Direct peer-to-peer mesh (WireGuard) |
| **Authentication** | HTTP headers (service tokens) | Network-level (ephemeral nodes + ACLs) |
| **Observability** | Cloudflare Access logs + LiteLLM logs | LiteLLM logs only (unless audit logging enabled) |
| **Portability** | Works from anywhere (no VPN client) | Requires Tailscale client/action |
| **Latency** | Routes through Cloudflare edge | Direct connection (usually lower latency) |
| **Cost** | Free tier available (limits apply) | Free for personal use |
| **FQDN support** | Yes (via tunnel config) | Localhost only for `serve` targets [web:107] |

**When to choose Cloudflare:**
- You already run Cloudflare Tunnel for other services
- You want centralized audit logs in Cloudflare dashboard
- You need to expose services bound to FQDNs (not localhost)
- You prefer HTTP-based auth over network-level VPN

**When to choose Tailscale:**
- You want lower latency (direct peer-to-peer)
- You prefer network-level segmentation
- Your services already bind to localhost
- You want simpler secret management (OAuth vs service tokens)

## Next Steps

### Option 1: Integrate into Production Workflow
Once this PoC passes, add the Cloudflare Access headers to all LiteLLM API calls in your production workflows:

```
env:
  LITELLM_HOST: ${{ secrets.LITELLM_HOST }}
  CF_ACCESS_CLIENT_ID: ${{ secrets.CF_ACCESS_CLIENT_ID }}
  CF_ACCESS_CLIENT_SECRET: ${{ secrets.CF_ACCESS_CLIENT_SECRET }}
run: |
  curl -X POST "https://$LITELLM_HOST/v1/chat/completions" \
    -H "CF-Access-Client-Id: $CF_ACCESS_CLIENT_ID" \
    -H "CF-Access-Client-Secret: $CF_ACCESS_CLIENT_SECRET" \
    -H "Content-Type: application/json" \
    -d '{"model": "gpt-4", "messages": [...]}'
```

### Option 2: Rotate Service Tokens
Set a reminder to rotate service tokens regularly:
1. Create a new service token
2. Update GitHub secrets
3. Revoke old token in Cloudflare dashboard

Consider implementing automated rotation with Bitwarden Secrets Manager or HashiCorp Vault. [web:149]

### Option 3: Switch to Block Mode
After several successful runs, review StepSecurity's egress audit logs and switch to `egress-policy: block` with an allowlist. [web:130]

### Option 4: Add More Services
Extend the tunnel config to expose additional services (like Karakeep RSS):

```
ingress:
  - hostname: litellm.yourdomain.com
    service: http://localhost:4000
  - hostname: rss.yourdomain.com
    service: http://karakeep.internal.example.com:8080
    path: /rss/*
  - service: http_status:404
```

Then create separate Access applications and service tokens for each service.

## Security Considerations

- **Tunnel Security:** Cloudflare Tunnel uses outbound-only connections, eliminating the need for inbound firewall rules or public IPs [web:15]
- **Service Token Lifespan:** Service tokens are long-lived by default; implement rotation policies [web:74]
- **Access Logs:** All authentication attempts are logged in Cloudflare Zero Trust for audit trails [web:74]
- **Secret Masking:** GitHub automatically redacts secret values from logs [web:153]
- **Egress Monitoring:** StepSecurity captures all outbound network traffic for audit [web:130]
- **Defense in Depth:** Even if GitHub secrets leak, Cloudflare Access logs show when/where the token was used

## Cost Considerations

**Cloudflare Zero Trust Free Tier includes:** [web:23]
- Up to 50 users
- Unlimited service tokens
- Basic Access policies
- 1 million requests/month

**Cloudflare Tunnel:** Free, unlimited usage

**StepSecurity Harden-Runner:** Free for public repositories (community tier) [web:128]

For private repositories or higher usage, review Cloudflare's pricing and StepSecurity's plans.

## References

- [Cloudflare Access Service Tokens](https://developers.cloudflare.com/cloudflare-one/access-controls/service-credentials/service-tokens/) [web:74]
- [Cloudflare Tunnel Docs](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/) [web:15]
- [GitHub Actions Secrets Best Practices](https://docs.github.com/actions/security-guides/using-secrets-in-github-actions) [web:153]
- [StepSecurity Harden-Runner Docs](https://docs.stepsecurity.io/harden-runner) [web:129]

## License

This is a proof-of-concept for secure CI/CD access to private infrastructure. Use as reference and adapt to your security requirements.
