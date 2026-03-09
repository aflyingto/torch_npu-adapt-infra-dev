# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an **infrastructure repository** for tracking and fixing PyTorch nightly compatibility issues in Ascend/pytorch (torch_npu). It is NOT a source code repository - the actual Ascend/pytorch code is cloned during CI from `Ascend/pytorch`.

**Purpose:**
- Track API compatibility issues between PyTorch nightly and Ascend/pytorch
- Store patches for fixing compatibility issues
- Provide skill documents for generating CI workflows
- Document common API break patterns and fixes

## Directory Structure

```
issues/     - Compatibility analysis reports (markdown)
patches/    - Git patches to fix issues (numbered .patch files)
skills/     - Skill documents for CI workflow generation
```

## Issue and Patch Naming Convention

Files use date-based numbering: `YYYY-MM-DD-NNN`

- `issues/YYYY-MM-DD-NNN-description.md` - Problem analysis
- `patches/NNNN-description.patch` - Corresponding fix (padded to 4 digits)

## Skill Documents

Skills in `skills/*/SKILL.MD` are instructions for generating GitHub Actions workflows:

| Skill | Output | Purpose |
|-------|--------|---------|
| `torch_npu-adapt-pytorch-nightly` | `.github/workflows/nightly-build.yml` | Validate PyTorch nightly compatibility |
| `torch_npu-build` | `.github/workflows/torch_npu-build.yml` | Build wheel with stable PyTorch |
| `auto-issues` | N/A | Submit issues to `aflyingto/torch_npu` via `gh` CLI |

**Key workflow patterns:**
- Use `git apply --directory=ascend_pytorch "$patch"` to apply patches
- Use `$GITHUB_ENV` for environment variables, not `export`
- Artifacts: `if: always()` for logs, `if: success()` for wheels

## Common API Break Patterns

When PyTorch upstream changes APIs, use these fix patterns:

| Break Type | Fix Pattern |
|------------|-------------|
| Struct member removed | Add module-level `static map + mutex` cache |
| Member renamed | Rename conflicting local type to avoid collision |
| Base class adds pure virtual | Implement override (return 0 if no semantic meaning) |
| Virtual signature changed | Update override, extract old logic to private helper |

## Patch Application

Patches are applied to cloned `Ascend/pytorch` source:

```bash
# Correct way - from repo root
git apply --directory=ascend_pytorch patches/0001-xxx.patch

# Wrong - do not cd into the directory
cd ascend_pytorch && git apply ../patches/xxx.patch  # DON'T DO THIS
```

## Target Repositories

- **This repo**: `aflyingto/torch_npu-adapt-infra-dev` - Infrastructure/tracking
- **Issues target**: `aflyingto/torch_npu` - Where issues are submitted
- **Source code**: `Ascend/pytorch` - Cloned during CI builds