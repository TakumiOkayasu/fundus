# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**fundus** - Murata Lab project (currently in initial setup phase)

This repository is newly initialized and does not yet contain application source code. Currently contains documentation for infrastructure setup.

## Existing Documentation

- `github-actions-self-hosted-runner-setup.md` - Guide for setting up GitHub Actions Self-hosted Runner on Proxmox VM for accelerating CI builds (e.g., kernel builds from 50-60min to 10-15min)

## Development Guidelines

When this project begins development:

1. Refer to `~/.claude/CLAUDE.md` for global Claude Code rules
2. Create feature branches before any code changes (never work on main)
3. Follow test-first development (RED-GREEN-REFACTOR)
4. Place temporary files in `claude_tmp/`
5. Place test files in `tests/`
