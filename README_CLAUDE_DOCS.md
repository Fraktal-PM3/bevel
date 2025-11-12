# Hyperledger Bevel - Claude Documentation

## Overview

This directory now contains comprehensive documentation for future Claude instances working with the Hyperledger Bevel codebase.

## Documentation Files

### 1. CLAUDE.md (Primary Reference)
- **Size:** 40KB, 1,266 lines
- **Purpose:** Comprehensive architecture guide covering all aspects of Bevel
- **Content:** 16 detailed sections with diagrams, code examples, and explanations
- **Best for:** In-depth understanding of architecture and design patterns

**What's Inside:**
1. Project Structure Overview
2. Platforms Architecture
3. Helm Charts Organization
4. Ansible Playbooks & Roles
5. Network.yaml Configuration
6. Shared Configuration Components
7. Build & Deployment Process
8. Testing Structure
9. Component Interaction Flows
10. Key Architectural Patterns
11. Configuration Reference
12. Extending Bevel
13. Troubleshooting Guide
14. Git Repository Structure
15. Quick Reference (paths and commands)
16. Summary

### 2. CLAUDE_INDEX.md (Quick Reference)
- **Size:** 13KB, 336 lines
- **Purpose:** Quick navigation and reference guide
- **Content:** Quick lookup tables, ASCII diagrams, critical paths
- **Best for:** Quick lookups and understanding the big picture

**What's Inside:**
- Quick navigation table to CLAUDE.md sections
- Core architecture pattern
- Directory tree structure
- Running commands
- 7-layer architecture breakdown
- Shared components checklist
- Platform comparison table
- Testing validation points
- Critical junctions explanation
- Troubleshooting links
- Next steps for future instances

## How to Use These Documents

### For First-Time Understanding
1. Read CLAUDE_INDEX.md - Quick Reference section first (5 minutes)
2. Read CLAUDE.md sections 1-5 (Project, Platforms, Helm, Ansible, Network.yaml)
3. Look at CLAUDE_INDEX.md architecture diagrams
4. Read CLAUDE.md sections 6-9 (Components, Deployment, Testing, Flows)

### For Specific Tasks
- **Deploying a network:** CLAUDE.md section 7 + CLAUDE_INDEX.md Running Commands
- **Debugging issues:** CLAUDE.md section 13 + CLAUDE_INDEX.md Troubleshooting
- **Understanding a file:** Use CLAUDE_INDEX.md "Critical Paths" + CLAUDE.md Quick Reference
- **Extending the code:** CLAUDE.md section 12 + Platform specifics

### For Reference
- File paths: CLAUDE.md section 15 and CLAUDE_INDEX.md Critical Paths
- Commands: CLAUDE_INDEX.md Running Commands
- Architecture overview: CLAUDE_INDEX.md Architecture Diagram
- Role patterns: CLAUDE.md section 4

## Key Concepts at a Glance

### The Core Chain
```
network.yaml → Ansible → Jinja2 templates → Generated values.yaml → 
Helm lint → Git commit → Flux → Kubernetes
```

### Critical Files
- `/platforms/shared/configuration/site.yaml` - Main orchestration
- `/platforms/network-schema.json` - Configuration validation
- `/platforms/shared/configuration/roles/` - Common logic for all platforms
- `/platforms/{platform}/charts/` - Kubernetes deployment specs
- `/platforms/{platform}/configuration/roles/` - Platform-specific logic

### The 7 Layers
1. **Config Input** - network.yaml
2. **Orchestration** - site.yaml + playbooks
3. **Role Logic** - Ansible roles
4. **Templates** - Jinja2 (.tpl files)
5. **Helm Charts** - K8s specs
6. **GitOps** - Generated values files in Git
7. **Kubernetes** - Running resources

## What Gets Documented

### Architecture Aspects Covered
- Overall project structure and directory layout
- How different DLT platforms (Fabric, Besu, etc.) are organized
- Helm chart organization and values generation pipeline
- Ansible playbooks and role structure
- Relationship between roles and Helm charts
- Network.yaml configuration structure and validation
- Shared configuration components and patterns
- Build and deployment process with detailed sequences
- Testing and validation points
- Component interaction flows with task breakdowns
- Architectural patterns and design decisions

### Specific Examples Included
- Fabric CA server deployment (full task breakdown)
- Peer node deployment with persistence
- Channel creation (Fabric 2.5.x)
- Vault authentication flow
- Multi-organization isolation
- GitOps integration with Flux

## Platform Coverage

Documentation covers all 7+ supported platforms:
- Hyperledger Fabric (extensive coverage)
- Hyperledger Besu
- Quorum
- Hyperledger Indy
- R3 Corda (OSS)
- R3 Corda Enterprise
- Substrate

Each follows the same shared architectural patterns documented here.

## For Future Claude Instances

These documents are designed to:
1. **Eliminate ramp-up time** - No need to re-explore the codebase
2. **Provide context** - Understand relationships between components
3. **Enable fast navigation** - Use CLAUDE_INDEX.md for quick lookup
4. **Support troubleshooting** - Section 13 and index have solutions
5. **Guide extensions** - Section 12 explains how to add new components
6. **Document patterns** - Section 10 explains architectural patterns

### Recommended Reading Order
1. This file (README_CLAUDE_DOCS.md) - Get oriented
2. CLAUDE_INDEX.md Quick Navigation - Find your topic
3. CLAUDE.md relevant section - Deep dive
4. CLAUDE.md section 15 (Quick Reference) - For commands and paths

### Using the Index
CLAUDE_INDEX.md has a navigation table at the top. Find your topic and go to that section of CLAUDE.md.

## Questions This Documentation Answers

### Understanding the Codebase
- Q: How is the project organized?
  A: See CLAUDE.md section 1, CLAUDE_INDEX.md Directory Tree

- Q: Which files should I read?
  A: See CLAUDE_INDEX.md "Most Important Files to Read First"

- Q: How do platforms differ?
  A: See CLAUDE.md section 2, CLAUDE_INDEX.md Platform Comparison

### Deployment & Operations
- Q: How does deployment work?
  A: See CLAUDE.md section 7, CLAUDE_INDEX.md "Understanding Each Layer"

- Q: What does network.yaml control?
  A: See CLAUDE.md section 5, CLAUDE_INDEX.md Configuration Hierarchy

- Q: How are components deployed?
  A: See CLAUDE.md section 9 for CA, peers, channels

- Q: How does GitOps work?
  A: See CLAUDE.md section 7.3, CLAUDE_INDEX.md Junction 3

### Extending the Code
- Q: How do I add a new component?
  A: See CLAUDE.md section 12.1

- Q: How do I add a new platform?
  A: See CLAUDE.md section 12.2

- Q: How do roles and charts work together?
  A: See CLAUDE.md sections 3 and 4

### Debugging & Troubleshooting
- Q: Pod stuck in Pending
  A: See CLAUDE.md section 13.1

- Q: Playbook failed
  A: See CLAUDE.md section 13.1

- Q: How do I verify deployment?
  A: See CLAUDE.md section 8.3

## Document Statistics

| Metric | Value |
|--------|-------|
| Total documentation | 53KB |
| Total lines | 1,602 |
| Sections in CLAUDE.md | 16 |
| Quick reference sections | 15+ |
| Diagrams included | 10+ |
| Code examples | 50+ |
| File paths documented | 50+ |
| Platform coverage | 7+ |
| Helm charts documented | 30+ |
| Ansible roles documented | 40+ |

## File Locations

All documentation is in the Bevel project root:

```
/home/adrian/Code/LTU/bevel/
├── CLAUDE.md                 (40KB - Primary reference)
├── CLAUDE_INDEX.md           (13KB - Quick reference)
└── README_CLAUDE_DOCS.md     (This file)
```

## Maintenance Notes

These documents represent the codebase as of 2025-11-12. If you're reading this significantly later:

1. The architecture patterns should still hold (they're foundational)
2. Specific file paths and role names may have changed
3. New platforms may have been added
4. The core chain (network.yaml → Ansible → Helm → Flux → Kubernetes) is likely still valid

For major updates, regenerate by:
1. Exploring `/platforms/` structure
2. Reading current playbooks
3. Checking role implementations
4. Reviewing sample network.yaml files

## Questions or Clarifications

If something in CLAUDE.md is unclear:
1. Check CLAUDE_INDEX.md - it has ASCII diagrams
2. Look at actual files in the codebase (paths are provided)
3. Review sample configurations in `/platforms/{platform}/configuration/samples/`
4. Examine actual role implementations in `/platforms/{platform}/configuration/roles/`

---

**Created:** 2025-11-12
**Status:** Ready for use by future Claude instances
**Completeness:** Very thorough (all requested focus areas covered)

Start with CLAUDE_INDEX.md and navigate to CLAUDE.md sections as needed.
