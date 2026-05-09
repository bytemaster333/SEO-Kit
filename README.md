# SEO Kit — Making Starknet Infrastructure Bulletproof

> **Ansible Collection:** `starknet_ops.node_toolkit` · Apache-2.0 · Ansible Core ≥ 2.15

[![CI](https://github.com/bytemaster333/SEO-Kit/actions/workflows/molecule.yml/badge.svg)](https://github.com/bytemaster333/SEO-Kit/actions/workflows/molecule.yml)

Professional Starknet node operators deserve the same hardened tooling that Ethereum
infrastructure has had for years. SEO Kit is the missing piece: a battle-tested Ansible
Collection that turns node deployment from a bespoke script collection into a reproducible,
observable, and self-healing system.

---

## The Problem We Solve

The Starknet network disruptions of September 2025 and January 2026 exposed a painful gap
in the ecosystem's operational maturity. During those incidents, independent node operators
faced a cascade of compounding failures:

- **Silent reorg drift** — nodes continued serving stale chain state without warning.
  Explorers and RPC consumers received data from a silently forked branch for minutes before
  operators noticed.
- **Corrupted snapshots** — several operators attempted emergency backups mid-reorg, producing
  archives that could not be used for recovery. Restoration took 12–36 hours of manual
  re-sync instead of the expected 30 minutes.
- **No automated triage** — operators had no standard tooling to distinguish "L1 RPC slow"
  from "my node is actually behind" from "the sequencer halted." Every diagnosis was manual.

None of these are hard engineering problems. They are **tooling problems**. SEO Kit exists
to close that gap permanently.

---

## Killer Features

### 1. Reorg-Aware Snapshots

Most backup tools treat a blockchain node like a database: stop the process, copy the
data directory, restart. This works until there is a chain reorganization — and then it
produces an archive that looks valid but contains a state that no longer exists on the
canonical chain.

SEO Kit's `snapshot` role runs a **cryptographic chain verification** before the service
is ever stopped:

```
starknet_blockNumber()
    └─ Fetch last N block headers via starknet_getBlockWithTxHashes()
           └─ Assert every block.status ∈ {ACCEPTED_ON_L2, ACCEPTED_ON_L1}
           └─ Assert block[i].parent_hash == block[i-1].block_hash  ← reorg tripwire
                    │
                    ├─ PASS → stop service → tar+zstd → upload S3 → restart service
                    │          Manifest written: block_number + block_hash + timestamp
                    │
                    └─ FAIL → ansible.builtin.fail() — no snapshot taken, no data loss
```

The verification depth is configurable (`snapshot_reorg_check_depth`, default: 10 blocks).
Every snapshot in S3 carries a manifest with its verified chain anchor, making restores
auditable: you know exactly which block your data represents.

### 2. L1-L2 Divergence Health Monitoring

Pathfinder and Juno nodes can fall behind the chain tip for several distinct reasons that
require different operator responses:

| Root Cause | Symptom | Correct Action |
|---|---|---|
| L1 RPC down or slow | Node stops processing L1 proofs | Fix L1 RPC endpoint |
| Network partition | Node has stale peers | Check P2P connectivity |
| Sequencer halt | All nodes equally behind | Wait; not an operator issue |
| Under-resourced host | Node processes blocks slowly | Scale compute |

SEO Kit's L1 health check (`l1_health_check.yml`) measures all axes of this problem in a
single pass:

1. **L1 RPC sanity** — `eth_blockNumber` with latency measurement and configurable
   `pathfinder_l1_latency_warn_ms` threshold (default: 2000 ms). High latency = impending
   proof processing backlog.
2. **L2 sync state** — `starknet_syncing` distinguishes "caught up" (`false`) from
   "syncing" (`{ current_block_num, highest_block_num }`).
3. **Drift thresholds** — configurable warn/fail gates (`pathfinder_l1_lag_warn_threshold:
   100`, `pathfinder_l1_lag_fail_threshold: 500`). The check fails the Ansible play before
   any harmful state is committed, not after.

All queries include **retry logic** (default 3 attempts, 5 s delay) so transient RPC
hiccups do not trigger false-positive failures in automated pipelines.

### 3. S3-First Disaster Recovery

When a node needs to be rebuilt — from a hardware failure, a data corruption event, or
simply provisioning a new region — the bottleneck is data transfer. Re-syncing Pathfinder
or Juno from genesis takes 20–60+ hours depending on network conditions.

SEO Kit's `recover.yml` playbook:

1. Stops the node service
2. Discovers the latest verified snapshot manifest from S3 (or local staging)
3. Streams the compressed archive directly to the data directory
4. Restarts the service already at block N, needing only a short sync tail

```bash
ansible-playbook playbooks/recover.yml \
  -i inventory/hosts.yml \
  -e recovery_node_service=pathfinder \
  --vault-id default@prompt
```

Snapshots are retained per `snapshot_s3_retention_count` (default: 5), giving a 5-point
rollback window. Compatible with AWS S3, Cloudflare R2, MinIO, and any S3-compatible
object store via `snapshot_s3_endpoint_url`.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    CONTROL PLANE (Ansible)                      │
│   site.yml ─── recover.yml ─── validate.yml                     │
└───────────────────────┬─────────────────────────────────────────┘
                        │ SSH / Ansible modules
          ┌─────────────▼─────────────────────────────┐
          │              TARGET HOST                   │
          │                                            │
          │  ┌──────────────────────────────────────┐  │
          │  │  Layer 1 — base role (OS hardening)  │  │
          │  │  • Non-root system users (pathfinder, juno)     │  │
          │  │  • sysctl: tcp_syncookies, somaxconn=65535      │  │
          │  │  • ulimits: nofile=65535 hard+soft              │  │
          │  │  • UFW: default-deny inbound, SSH rate-limit    │  │
          │  └─────────────────┬────────────────────┘  │
          │                    │                        │
          │  ┌─────────────────▼────────────────────┐  │
          │  │  Layer 2 — node role (pathfinder|juno)│  │
          │  │  • Pinned binary, SHA256-verified      │  │
          │  │  • Systemd unit with security hardening│  │
          │  │  • .env (EnvironmentFile, mode 0640)   │  │
          │  │  • UFW: open RPC/P2P ports             │  │
          │  │  • L1-L2 health check (pre-deploy)     │  │
          │  └─────────────────┬────────────────────┘  │
          │                    │                        │
          │  ┌─────────────────▼────────────────────┐  │
          │  │  Layer 3 — snapshot role (ops)        │  │
          │  │  • Reorg check → archive → manifest   │  │
          │  │  • S3 upload + retention pruning      │  │
          │  │  • Cron-scheduled (default 03:00)     │  │
          │  └─────────────────┬────────────────────┘  │
          └─────────────────────────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │   S3-Compatible     │
                    │  Object Storage     │
                    │  (AWS / R2 / MinIO) │
                    └─────────────────────┘
```

Each layer is independently deployable. The `base` role runs once at host provisioning;
`pathfinder` or `juno` manage the node lifecycle; `snapshot` runs as a cron-driven ops
task. Any role can be applied in isolation via Ansible tags.

---

## Quick Start

### Prerequisites

- Ansible Core ≥ 2.15 and Python 3.10+ on the control node
- Target: Ubuntu 22.04 LTS or Debian 12
- An L1 Ethereum RPC endpoint (Alchemy, Infura, or self-hosted)

### Step 1 — Install the collection

```bash
ansible-galaxy collection install git+https://github.com/bytemaster333/SEO-Kit.git
# or from Galaxy once published:
ansible-galaxy collection install starknet_ops.node_toolkit
```

### Step 2 — Configure your inventory

```yaml
# inventory/hosts.yml
all:
  children:
    pathfinder_nodes:
      hosts:
        node-01.example.com:
          ansible_user: ubuntu

# inventory/group_vars/pathfinder_nodes.yml
pathfinder_version: "0.14.3"
pathfinder_network: mainnet
pathfinder_ethereum_url: "{{ vault_ethereum_url }}"   # stored in ansible-vault

# Snapshot to S3
snapshot_s3_enabled: true
snapshot_s3_bucket: "my-starknet-snapshots"
snapshot_s3_region: "eu-central-1"
snapshot_s3_access_key: "{{ vault_s3_access_key }}"   # ansible-vault
snapshot_s3_secret_key: "{{ vault_s3_secret_key }}"   # ansible-vault
```

```bash
# Encrypt your secrets
ansible-vault create inventory/group_vars/vault.yml
```

### Step 3 — Deploy

```bash
# Full stack: harden OS + deploy node + configure snapshots
ansible-playbook playbooks/site.yml \
  -i inventory/hosts.yml \
  --vault-id default@prompt

# Emergency recovery (restores from latest S3 snapshot)
ansible-playbook playbooks/recover.yml \
  -i inventory/hosts.yml \
  --vault-id default@prompt
```

---

## Role Reference

### `base`

| Variable | Default | Description |
|---|---|---|
| `base_service_users` | `[pathfinder, juno]` | Non-root system users to create |
| `base_sysctl_settings` | See defaults | Kernel tuning parameters |
| `base_ufw_allowed_ports` | `[22/tcp rate-limit]` | Inbound firewall allow rules |
| `base_ufw_enabled` | `true` | Enable UFW after rule configuration |

### `pathfinder`

| Variable | Default | Description |
|---|---|---|
| `pathfinder_version` | `"0.14.3"` | Pinned binary version |
| `pathfinder_network` | `mainnet` | `mainnet` or `sepolia` |
| `pathfinder_ethereum_url` | `""` | L1 RPC URL — **must be vault-encrypted** |
| `pathfinder_rpc_port` | `9545` | HTTP-RPC listen port |
| `pathfinder_health_check_enabled` | `true` | Run L1-L2 divergence check |
| `pathfinder_l1_lag_warn_threshold` | `100` | Warn at this many blocks of drift |
| `pathfinder_l1_lag_fail_threshold` | `500` | Abort play at this drift level |

### `juno`

| Variable | Default | Description |
|---|---|---|
| `juno_version` | `"0.12.5"` | Pinned binary version |
| `juno_network` | `mainnet` | `mainnet` or `sepolia` |
| `juno_ethereum_url` | `""` | L1 RPC URL — **must be vault-encrypted** |
| `juno_rpc_port` | `6060` | HTTP-RPC listen port |
| `juno_p2p_port` | `7777` | libp2p listen port |

### `snapshot`

| Variable | Default | Description |
|---|---|---|
| `snapshot_mode` | `backup` | `backup` or `restore` |
| `snapshot_reorg_check_depth` | `10` | Blocks to verify before backup |
| `snapshot_s3_enabled` | `false` | Enable S3 upload |
| `snapshot_s3_bucket` | `""` | Target S3 bucket name |
| `snapshot_s3_retention_count` | `5` | Snapshots to keep in S3 |
| `snapshot_cron_enabled` | `true` | Schedule backup via cron |
| `snapshot_cron_hour` | `"3"` | Cron hour (UTC) |

> **Security:** `snapshot_s3_access_key`, `snapshot_s3_secret_key`, and
> `pathfinder_ethereum_url` must always be loaded from an `ansible-vault` encrypted file.
> Never commit secrets to your inventory repository.

---

## Testing

```bash
pip install -r requirements.txt          # molecule, ansible-lint, yamllint

# Run full Molecule test sequence for each role
cd roles/base       && molecule test
cd roles/pathfinder && molecule test
cd roles/juno       && molecule test
cd roles/snapshot   && molecule test

# Lint the entire collection
ansible-lint roles/
yamllint .

# Build the Galaxy artifact
ansible-galaxy collection build --force
```

CI runs the full matrix (ubuntu2204 × debian12 × all 4 roles) on every push via
GitHub Actions. See `.github/workflows/molecule.yml`.

---

## Security Model

| Concern | Mitigation |
|---|---|
| Secret exposure in logs | `no_log: true` on credential tasks; EnvironmentFile mode 0640 |
| Node privilege escalation | `NoNewPrivileges=yes`, `ProtectSystem=strict` in systemd units |
| L1 RPC URL in process list | Passed via EnvironmentFile, not CLI args |
| Network attack surface | UFW default-deny; only declared ports opened |
| Reorg data corruption | Snapshot aborts on any chain integrity failure |

---

## Roadmap

| Milestone | Target | Description |
|---|---|---|
| **v0.1** (current) | Q2 2026 | Base, pathfinder, juno, snapshot roles + CI; **S3 retention pruning** for backup cost control (`snapshot_s3_retention_count`) |
| **v0.2 — Validator Support** | Q3 2026 | Starknet full-node validator configuration; key management via HashiCorp Vault |
| **v0.3 — HA / Failover** | Q3 2026 | Multi-node inventory patterns; automatic failover playbook using `recover.yml` |
| **v0.4 — Terraform Integration** | Q4 2026 | AWS / GCP / Hetzner Terraform modules that provision hosts and invoke this collection |
| **v0.5 — Grafana Stack** | Q4 2026 | Pre-built dashboards for Pathfinder Prometheus metrics; alert rules for drift thresholds |
| **v1.0 — Galaxy Publish** | Q1 2027 | Ansible Galaxy publication; semantic versioning with full backwards compatibility |

---

## Contributing

Contributions are welcome. Please open an issue before submitting a large PR so the scope
can be discussed. All roles must pass `ansible-lint` (0 failures) and `molecule test` before
merge.

## License

[Apache-2.0](LICENSE)

---

*Built by [bytemaster333](https://github.com/bytemaster333) for the Starknet ecosystem.*
