# AWS MGN Migration Guide

Complete end-to-end guide for migrating on-premises servers to AWS using AWS Application Migration Service (MGN) with AWS Transform.

## Documents

| File | Description |
|------|-------------|
| [mgn-complete-migration-workflow.md](mgn-complete-migration-workflow.md) | Full workflow with architecture diagrams, state machines, and best practices |
| [mgn-migration-steps-sequence.md](mgn-migration-steps-sequence.md) | Sequential CLI commands for each step |
| [mgn-migration-troubleshooting-blog.md](mgn-migration-troubleshooting-blog.md) | Troubleshooting blog post — all issues encountered and resolved |

## What's Covered

- AWS Control Tower Landing Zone setup (v3.3 → v4.0 upgrade)
- AWS Organizations OU structure for governance
- MGN Connector (agentless discovery from vCenter)
- MGN Replication Agent installation and configuration
- Replication configuration and launch template setup
- Test migration and validation
- Final cutover and finalization
- Common challenges and resolutions

## Migration Timeline

From discovery to cutover complete: ~3 hours (including troubleshooting)

## Author

Ramandeep Chandna
