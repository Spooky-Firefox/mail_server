# mail-server

[docker-mailserver](https://github.com/docker-mailserver/docker-mailserver) for `ronstad.se`,
deployed on the `olle-deb-server` k3s cluster via ArgoCD, using the community
[docker-mailserver-helm](https://github.com/docker-mailserver/docker-mailserver-helm) chart.

## Before you go further: read this

- **Your public IP's reverse DNS is `c-92-35-180-204.bbcust.telenor.se`** - a
  generic residential PTR you don't control. Many providers (Gmail, Microsoft)
  hard-reject or heavily spam-score mail arriving from IPs that look
  residential, regardless of correct SPF/DKIM/DMARC. This setup will work for
  receiving mail and for sending to less strict destinations, but don't expect
  guaranteed inbound delivery to Gmail/Outlook without a static IP on a
  business-class line (or relaying outbound mail through a smarthost - see
  `DEFAULT_RELAY_HOST` in `helm/values.yaml` if this becomes a problem).
- **Telenor may block outbound port 25** on residential connections. Test this
  before relying on direct sending (e.g. `nc -vz smtp.gmail.com 25` from the
  cluster's node, or ask Telenor).
- This deploys **one replica only** - docker-mailserver doesn't support
  horizontal scaling (single mailbox storage, no clustering).

## What this repo contains

```
helm/values.yaml       - Helm values for the docker-mailserver chart
k8s/certificate.yaml   - cert-manager Certificate for mail.ronstad.se
argocd/application.yaml - the ArgoCD Application tying it together
```

`mail.ronstad.se` already resolves to `92.35.180.204` (wildcard DNS covers all
of `*.ronstad.se`), and Traefik/cert-manager (`letsencrypt-prod` ClusterIssuer)
/ the `local-path` StorageClass are already set up on the cluster - reused
here as-is. Mail traffic (SMTP/IMAP) does **not** go through Traefik: it's
exposed directly via k3s's built-in ServiceLB as its own `LoadBalancer`
Service, bound straight to the node's LAN IP (`192.168.1.136`), the same way
Traefik itself gets `192.168.1.136` for 80/443.

## Setup steps

### 1. Create the GitHub repo and push

```console
cd /home/olle/Documents/mail_server
git init -b main
git add helm k8s argocd README.md
git commit -m "Add docker-mailserver k8s/Helm/ArgoCD setup"
# create github.com/Spooky-Firefox/mail_server first (empty repo, no README/license), then:
git remote add origin git@github.com:Spooky-Firefox/mail_server.git
git push -u origin main
```

### 2. Forward router ports

Forward these from your WAN interface to `192.168.1.136` (same box/ports, no
NAT rewrite) - the same way 80/443 are already forwarded for Traefik:

| Port | Protocol |
| ---- | -------- |
| 25   | SMTP (inbound mail from other servers) |
| 465  | SMTP submission (implicit TLS, for mail clients) |
| 587  | SMTP submission (STARTTLS, for mail clients) |
| 993  | IMAPS (mail clients) |

### 3. Apply the ArgoCD Application

Once the repo is pushed:

```console
kubectl apply -f argocd/application.yaml
kubectl -n argocd get application mail-server -w
```

ArgoCD will create the `mail` namespace, request the TLS certificate, and
deploy the chart. `kubectl -n mail get pods,pvc,svc` to check progress.
`kubectl -n mail get svc mail-server-docker-mailserver` should show
`192.168.1.136` as `EXTERNAL-IP` once the LoadBalancer settles (chart-derived
service name - check the actual name it renders).

### 4. DNS records to add at your DNS provider for `ronstad.se`

| Record | Value |
| ------ | ----- |
| `MX` on `ronstad.se` | `10 mail.ronstad.se.` |
| `TXT` on `ronstad.se` (SPF) | `v=spf1 mx ~all` |
| `TXT` on `_dmarc.ronstad.se` | `v=DMARC1; p=none; rua=mailto:postmaster@ronstad.se` |
| `TXT` DKIM | generated in step 6 below, add once you have it |

Start DMARC at `p=none` (monitor only) and tighten to `quarantine`/`reject`
once you've confirmed mail flows correctly for a while.

### 5. Create a mailbox

```console
kubectl exec -it -n mail deploy/mail-server-docker-mailserver -- \
  setup email add you@ronstad.se yourpassword
```

(deployment name depends on how the chart renders `fullnameOverride` -
`kubectl -n mail get deploy` to confirm the exact name.)

### 6. Generate and publish DKIM

```console
kubectl exec -it -n mail deploy/mail-server-docker-mailserver -- \
  setup config dkim keysize 2048 domain 'ronstad.se'

kubectl exec -it -n mail deploy/mail-server-docker-mailserver -- \
  cat /tmp/docker-mailserver/rspamd/dkim/ronstad.se.mail.txt
```

Publish the printed value as a `TXT` record at `mail._domainkey.ronstad.se`
(selector name is `mail` by default).

### 7. Test

- `openssl s_client -connect mail.ronstad.se:993` - check the cert chains to
  Let's Encrypt and matches `mail.ronstad.se`.
- Send a test mail to a [mail-tester.com](https://www.mail-tester.com)
  address and check the score - this will surface any remaining SPF/DKIM/
  DMARC/RBL issues, including the residential-IP problem flagged above.

## Notes

- Rspamd handles spam filtering and DKIM signing (chart defaults:
  `ENABLE_RSPAMD=1`, `ENABLE_OPENDKIM=0`) - this is the combination
  docker-mailserver's own docs recommend.
- `deployment.strategy.type: Recreate` (chart default) - required since only
  one pod can hold the `ReadWriteOnce` PVCs at a time; expect ~30-60s of
  downtime on every rollout.
- Chart version pinned via git tag `docker-mailserver-5.1.1` in
  `argocd/application.yaml` - bump deliberately, check the chart's release
  notes for breaking changes first.
