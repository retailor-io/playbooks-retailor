# elk_stack

Deploys Elasticsearch, Logstash and Kibana as a single Docker Compose stack
on a Debian host that already has Docker (and the `docker compose` plugin)
installed and running. All three services join a shared bridge network
(`elk-network` by default) so they can reach each other by container/service
name — no hardcoded IPs, no `--link`.

## Requirements

- Debian host with Docker Engine + Docker Compose plugin already installed
- Ansible collection: `community.docker` (`ansible-galaxy collection install community.docker`)
- Python `docker` + `requests` modules on the target (the role installs these via pip)
- Enough RAM for the JVM heaps you configure (Elasticsearch + Logstash default to ~1.5G combined; leave headroom for OS/Docker)

## What it does

1. Verifies Docker and Docker Compose are usable.
2. Creates `{{ elk_base_dir }}` (default `/opt/elk`) with a `config/` tree.
3. Templates:
   - `config/elasticsearch/elasticsearch.yml`
   - `config/kibana/kibana.yml`
   - `config/logstash/logstash.yml`, `pipelines.yml`, and three pipeline
     configs: `pipeline/main.conf` (generic beats/tcp/udp), `pipeline/pipeline-nginx.conf`
     (nginx access logs), `pipeline/pipeline-laravel.conf` (Laravel logs)
   - `docker-compose.yml` — one file, three services, one shared network, a named volume for ES data
4. Brings the stack up with `docker compose up`.
5. Waits for Elasticsearch to respond, then (if security is enabled) sets the
   `kibana_system` and `logstash_system` built-in user passwords via the
   Elasticsearch security API, and recreates Kibana/Logstash so they pick up
   working credentials.
6. Waits for Kibana's `/api/status` to come back healthy.

## Networking

All three containers run on the `elk_network_name` bridge network
(default `elk-network`, subnet `172.28.0.0/24`). Inside that network, all
inter-service traffic uses `http://` or `https://` consistently based on
`elk_tls_enabled` (default stack, TLS on):

- Elasticsearch is reachable at `https://elasticsearch:9200`
- Kibana is configured with `elasticsearch.hosts: ["https://elasticsearch:9200"]`
- Logstash's pipelines all point at `https://elasticsearch:9200`

With `elk_tls_enabled: false`, all three of the above use `http://` instead.

Host ports (`elk_elasticsearch_http_port`, `elk_kibana_http_port`,
`elk_logstash_beats_port`, `elk_logstash_nginx_beats_port`,
`elk_logstash_laravel_beats_port`, etc.) are only for reaching the stack
*from outside* — the containers themselves never use `localhost` to talk to
each other.

## Security

`elk_security_enabled: true` by default (recommended). This turns on
Elastic's built-in security (basic auth, no TLS between containers since
traffic stays on the internal Docker network). Change these before running
in anything but a scratch environment — ideally via Ansible Vault:

```yaml
elk_elastic_password: "..."
elk_kibana_system_password: "..."
elk_logstash_system_password: "..."
elk_kibana_encryption_key: "..."   # 32+ random chars, e.g. `openssl rand -hex 32`
```

Set `elk_security_enabled: false` for a quick, unauthenticated local/dev
stack (no passwords) — do not do this on anything reachable from untrusted
networks.

**Note:** `elk_tls_enabled` and `elk_security_enabled` are independent
switches you can set separately, but Elasticsearch only applies TLS
(`xpack.security.http.ssl.*`) when X-Pack security itself is on. So if
`elk_tls_enabled: true` and `elk_security_enabled: false`, this role
force-enables security anyway purely to make TLS possible — `elastic`,
`kibana_system`, and `logstash_system` all get real passwords from
`elk_elastic_password`/`elk_kibana_system_password`/
`elk_logstash_system_password` in that mode too, exactly as if
`elk_security_enabled: true` had been set, so change those from their
placeholder defaults the same way you would for a normal secured stack.
Setting both `elk_tls_enabled: false` and `elk_security_enabled: false` is
the only way to get a fully open, unauthenticated, plain-HTTP stack.

## Example playbook

```yaml
- hosts: elk_servers
  become: true
  roles:
    - role: elk_stack
      vars:
        elk_version: "8.15.3"
        elk_elasticsearch_heap_size: "2g"
        elk_logstash_heap_size: "1g"
        elk_elastic_password: "{{ vault_elk_elastic_password }}"
        elk_kibana_system_password: "{{ vault_elk_kibana_password }}"
        elk_logstash_system_password: "{{ vault_elk_logstash_password }}"
        elk_kibana_encryption_key: "{{ vault_elk_kibana_enc_key }}"
```

## Customizing the Logstash pipeline

Three pipelines run side by side inside the same Logstash process, each on
its own beats port (they can't share a port):

| Pipeline | Template | Beats port var | Purpose |
|---|---|---|---|
| `main` | `main.conf.j2` | `elk_logstash_beats_port` (5044) | Generic beats input, plus raw JSON over TCP/UDP (5000) |
| `nginx_pipeline` | `pipeline-nginx.conf.j2` | `elk_logstash_nginx_beats_port` (5045) | nginx access log parsing (grok + geoip) |
| `laravel_pipeline` | `pipeline-laravel.conf.j2` | `elk_logstash_laravel_beats_port` (5046) | Laravel log ingestion |

All three ship to Elasticsearch using the same `elk_tls_enabled`/
`elk_security_enabled` logic as the rest of the stack (TLS scheme + the
`logstash_system` credentials, when applicable). To add your own
filters/inputs/outputs, either edit these templates directly, or drop an
additional `*.conf.j2` file into `templates/` and extend `pipelines.yml.j2`
(with a new `pipeline.id`/`path.config` entry and a beats port that doesn't
collide with the ones above) and `tasks/main.yml` (a matching `template:`
task) to deploy it.

## Idempotency notes

- Re-running the role only recreates containers whose config actually
  changed (via handlers), plus a compose reconciliation pass.
- Setting the `kibana_system`/`logstash_system` passwords is safe to repeat;
  Elasticsearch just resets them to the same value each time and the role
  only recreates those two containers when the API call reports a change.

## TLS (public certs for a private-IP node)

`elk_tls_enabled: true` (default) gets Elasticsearch, Kibana, and Logstash a
**real, publicly-trusted certificate** for a DNS name that happens to point
at a private IP (e.g. `logs-eu.domain.com -> 10.0.99.5`), using the ACME
**DNS-01** challenge against Cloudflare. This works even though the host
itself is unreachable from the internet, because DNS-01 only requires the
ability to create a `_acme-challenge` TXT record — no inbound HTTP/HTTPS
access to the node is needed at all.

### Why DNS-01, and why one shared cert

- Public CAs don't issue certificates for bare IP SANs (especially not
  RFC1918/private addresses) — the workaround is a cert for a **DNS name**
  that resolves to the private IP, with clients connecting via that name.
- Since all three services live on the same node behind the same DNS name,
  they share **one cert** rather than one per service. Elastic explicitly
  supports (and this is a common pattern) using [a single shared HTTP CA
  across Elasticsearch, Kibana, and other stack components](https://www.elastic.co/docs/deploy-manage/deploy/self-managed/tutorial-self-managed-secure)
  — separate CAs/certs per component exist to isolate trust domains, which
  isn't a goal here since everything is already inside the same trust
  boundary (one box, one operator, one Docker network).
- Because the cert comes from a **public** CA (Let's Encrypt, via lego's
  default ACME server), Kibana and Logstash don't need a custom CA
  certificate distributed to them to trust Elasticsearch's cert — their
  default trust stores already trust Let's Encrypt's root. This removes an
  entire category of "distribute the CA cert everywhere" complexity that
  self-signed/internal-CA setups require.

### What this role does

1. Expects `elk_cloudflare_token_env_file` (default
   `/etc/lego/cloudflare-token.env`) to **already exist** on the box,
   containing `CF_DNS_API_TOKEN=...`, root-owned, mode `0600`. This role
   only *reads* that file — it never receives, logs, or stores the token
   itself. See "IaC responsibilities" below for where this file comes from.
2. Installs the [`lego`](https://github.com/go-acme/lego) ACME client (a
   single static Go binary — no Python/snap dependencies, easy to pin an
   exact version).
3. On first run, issues a certificate for `elk_tls_domain` via `lego ... run`,
   landing at `{{ elk_lego_certs_dir }}/{{ elk_tls_domain }}.{crt,key,issuer.crt}`.
4. Mounts that directory read-only into all three containers and wires
   `xpack.security.http.ssl.*` (Elasticsearch), `server.ssl.*` (Kibana), and
   `ssl_enabled` (Logstash's elasticsearch output) to point at it.
5. Installs a `lego-renew.service` + `lego-renew.timer` systemd pair that
   runs daily, checks expiry, and **only** re-issues (and only then restarts
   the three containers, via lego's `--run-hook`) when the cert is within
   `elk_tls_renew_days` of expiring. No-op checks never restart anything.

### IaC responsibilities (Pulumi, not this role)

This role deliberately does **not** manage Cloudflare DNS records, zones, or
API tokens — that's Pulumi's job, since it already holds the Cloudflare
credentials and provisions the box. The pseudocode below sketches what
Pulumi needs to set up per node:

```python
# Pulumi pseudocode - see conversation history / infra repo for the real version
apex_zone = cloudflare.get_zone(name="domain.com")

# A record already exists in this setup: logs-eu.domain.com -> 10.0.99.x
# (proxied=False - a private IP can't sit behind Cloudflare's proxy/edge)

# Scoped API token: Zone.DNS.Edit, restricted to the apex zone's ID (or a
# delegated child zone, e.g. logs-eu.domain.com as its own zone, for harder
# isolation). This is the tightest scope Cloudflare's token model supports -
# there's no way to restrict a token to a single record name/pattern.
acme_dns_token = cloudflare.ApiToken(
    "logs-eu-acme-dns-token",
    policies=[{
        "permission_groups": [dns_edit_permission_group],
        "resources": {f"com.cloudflare.api.account.zone.{apex_zone.id}": "*"},
    }],
    # optional: condition.request_ip.in_ to pin the token to the node's
    # egress IP, and/or expires_on for periodic rotation via Pulumi re-runs
)

# Hand the token to the VM via cloud-init, written root-only - Pulumi never
# passes this to Ansible as a variable.
cloud_init_write_files = [{
    "path": "/etc/lego/cloudflare-token.env",
    "owner": "root:root",
    "permissions": "0600",
    "content": f"CF_DNS_API_TOKEN={acme_dns_token.value}\n",
}]
```

For stronger isolation than a single shared zone token, NS-delegate
`logs-eu.domain.com` (and `logs-us.domain.com`, etc.) to their own
Cloudflare zones, and scope each region's token to only its own zone ID —
that way a compromised `logs-eu` node's token can't touch `logs-us` or the
apex domain's records at all.

### Key variables

| Variable | Purpose |
|---|---|
| `elk_tls_enabled` | Master switch; `false` reverts to plain HTTP |
| `elk_tls_domain` | The DNS name (e.g. `logs-eu.domain.com`) the cert covers and clients must connect via |
| `elk_tls_acme_email` | Contact address registered with the ACME account |
| `elk_tls_acme_server` | ACME endpoint — point at Let's Encrypt **staging** while testing to avoid production rate limits |
| `elk_cloudflare_token_env_file` | Path to the pre-existing, cloud-init-provisioned token file |
| `elk_tls_renew_days` | Renew when fewer than this many days remain before expiry |
| `elk_lego_version` | Pinned lego release to install |

### Operational notes

- **Testing**: set `elk_tls_acme_server` to
  `https://acme-staging-v02.api.letsencrypt.org/directory` first — staging
  certs aren't publicly trusted but validate the whole DNS-01/Cloudflare
  flow without touching Let's Encrypt's production rate limits.
- **Downtime on renewal**: renewal restarts all three containers via
  `docker compose restart`, which is brief (~10-20s) but not zero-downtime.
  Acceptable for a single-node stack; a zero-downtime rolling reload would
  require a multi-node Elasticsearch cluster, which is out of scope here.
- **Clients must use the DNS name, not the raw IP** — connecting to
  `https://10.0.99.5:9200` directly will fail certificate hostname
  verification even though the cert is otherwise valid, since the cert's
  SAN is `logs-eu.domain.com`, not the IP.
