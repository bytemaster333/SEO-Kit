# Changelog

## [0.1.0] - Unreleased

### Added
- Initial collection skeleton: `starknet_ops.node_toolkit`
- `base` role: OS hardening (sysctl, ulimits, UFW, non-root users)
- `pathfinder` role: Pinned binary install, systemd service, L1-L2 divergence health check
- `juno` role: Go binary install, P2P & RPC configuration, systemd service
- `snapshot` role: Reorg-aware backup with `starknet_getBlockWithTxHashes` chain verification, S3 upload
- `recover.yml` playbook: Crisis recovery from latest healthy snapshot
- Molecule test matrix: Ubuntu 22.04 + Debian 12 for all roles
- GitHub Actions CI workflow
