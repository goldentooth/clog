# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview
This is a mdBook project documenting the construction and configuration of a Raspberry Pi Kubernetes cluster. The content is written in Markdown and generates a static website using mdBook.

## Common Commands

### Build the book
```bash
mdbook build
```

### Serve locally for development (auto-rebuilds on changes)
```bash
mdbook serve
```

### Clean build artifacts
```bash
mdbook clean
```

## Project Structure

### Content Organization
- `src/` - All content files
  - `SUMMARY.md` - Table of contents and chapter ordering
  - `001_*.md` through `048_*.md` - Individual chapters
  - `images/` - Screenshots and diagrams referenced in chapters

### Theme Customization
- `theme/` - Custom theme overrides
  - `index.hbs` - HTML template
  - `css/chrome.css` - Style customizations
  - `book.js` - JavaScript functionality

### Chapter Naming Convention
Chapters use a three-digit prefix pattern: `XXX_chapter_name.md`
- Example: `001_building_the_pi_bramble.md`
- New chapters should continue the sequence (next would be `049_*.md`)

## Writing New Content

When adding new chapters:
1. Create the file following the naming pattern in `src/`
2. Add an entry to `src/SUMMARY.md` to include it in the table of contents
3. Place any images in `src/images/` and reference with relative paths

## Deployment
The book automatically deploys to GitHub Pages via GitHub Actions when changes are pushed to the main branch. The live site is available at https://clog.goldentooth.net/

## Content Focus
This project documents:
- Kubernetes cluster setup on Raspberry Pi hardware
- Infrastructure components (HAProxy, MetalLB, ArgoCD, cert-manager)
- GitOps practices and configurations
- Observability tools (Prometheus, Grafana, Loki)
- Platform engineering and distributed systems concepts

## Goldentooth CLI Tool

The author has a custom CLI tool called `goldentooth` for managing the Raspberry Pi cluster. This tool is located at `/usr/local/bin/goldentooth` and provides a unified interface for cluster operations.

### Key Commands
```bash
# Run commands on cluster nodes
goldentooth command <host/group> '<command>'
goldentooth command_raw <host/group> '<command>'  # For complex commands with special chars
goldentooth command_root <host/group> '<command>' # Run as root

# Examples
goldentooth command allyrion 'hostname'
goldentooth command_raw allyrion 'dd if=/dev/zero of=/mnt/shared/test bs=1M count=1024'
goldentooth ping all_pis
goldentooth uptime bettley,cargyll
```

### Available Host Groups
- `all` - All cluster nodes
- `all_pis` - All Raspberry Pi nodes (excluding non-Pi hardware)
- `k8s_control_plane` - Kubernetes control plane nodes
- `k8s_workers` - Kubernetes worker nodes
- Individual hostnames: `allyrion`, `bettley`, `cargyll`, etc.

### Playbook Commands
The tool automatically exposes Ansible playbooks as commands:
- `goldentooth bootstrap_k8s` - Bootstrap Kubernetes cluster
- `goldentooth setup_zfs` - Setup ZFS storage
- `goldentooth setup_ssh_certificates` - Configure SSH certificates
- And many more based on playbooks in the ansible repository

When documenting command outputs or testing cluster functionality, you can use this tool instead of assuming SSH access.

## Important Notes
- This is a documentation project with no application code or tests
- The project uses mdBook v0.4.37 (as specified in CI)
- MathJax is enabled for mathematical notation
- The book has extensive future topics planned (see comments in SUMMARY.md)