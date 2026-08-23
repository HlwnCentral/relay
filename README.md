# Paopaogou → GitHub Actions → Cloudflare → Pawdoll

This package avoids the problem where the Paopaogou subscription server returns
an empty response to Cloudflare-originated requests.

Flow:

```text
Paopaogou subscription
        ↓
GitHub Actions (every 15 minutes)
        ↓
relay/subscription.txt
        ↓
Cloudflare Worker
        ↓
Pawdoll-compatible sing-box JSON
        ↓
Pawdoll
```

## 1. Create a GitHub repository

Create a new repository and upload the contents of this package to it.

For the simplest Cloudflare setup, make the repository **public**. The actual
subscription URL/token is still stored as a GitHub Actions secret and is not
committed to the workflow file.

## 2. Add the subscription URL as a GitHub secret

In the repository open:

**Settings → Secrets and variables → Actions → New repository secret**

Name:

```text
SUBSCRIPTION_URL
```

Value:

```text
https://45.78.78.177:30322/v2b/paopaogou/api/v1/client/subscribe?token=YOUR_TOKEN
```

You can instead use the HTTP IP endpoint if that is the one that works from
GitHub Actions.

## 3. Allow GitHub Actions to push updates

Open:

**Settings → Actions → General → Workflow permissions**

Choose:

**Read and write permissions**

Save.

## 4. Run the relay once manually

Open:

**Actions → Refresh Paopaogou Subscription → Run workflow**

After it succeeds, the repository should contain:

```text
relay/subscription.txt
```

The Action then checks for updates every 15 minutes and commits only when the
subscription changes.

## 5. Configure the Cloudflare Worker

Open:

```text
cloudflare-worker/wrangler.toml
```

Change:

```toml
UPSTREAM_URL = "https://raw.githubusercontent.com/USERNAME/REPOSITORY/main/relay/subscription.txt"
```

to your actual GitHub username and repository.

Example:

```toml
UPSTREAM_URL = "https://raw.githubusercontent.com/example/pawdoll-relay/main/relay/subscription.txt"
```

## 6. Deploy the Worker

From the `cloudflare-worker` folder:

```powershell
npm install -g wrangler
wrangler login
wrangler deploy
```

Wrangler will give you a URL such as:

```text
https://pawdoll-paopaogou-github-relay.YOUR-SUBDOMAIN.workers.dev
```

## 7. Add it to Pawdoll

Create a **Remote** profile with:

```text
https://pawdoll-paopaogou-github-relay.YOUR-SUBDOMAIN.workers.dev/singbox
```

A 60-minute Pawdoll refresh interval is fine. GitHub refreshes the source every
15 minutes, while the Worker fetches the latest mirrored copy.

## Worker behavior

The Cloudflare Worker keeps the working Pawdoll configuration used previously:

- direct Android/system DNS
- IPv4-only TUN
- `auto-fast`
- manual node selection
- VLESS, VMess, Trojan, Shadowsocks, Hysteria2/Hy2 conversion
- Pawdoll/newer sing-box-compatible configuration
- no custom `_meta` field
- no legacy inbound `sniff` field

## Security note

Do not put the provider subscription URL/token directly in the workflow YAML.
Keep it in the `SUBSCRIPTION_URL` GitHub Actions secret.

If the GitHub repository is public, the mirrored `relay/subscription.txt` will
also be public. If the provider treats the node list itself as sensitive, use a
private relay approach instead of publishing the mirror.
