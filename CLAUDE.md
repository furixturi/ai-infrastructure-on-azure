# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# Azure HPC GPU Cluster Operations

This repo contains infrastructure validation tests, deployment references, AI training examples, and operational knowledge for running GPU workloads on Azure. It supports multiple orchestrators:

- **[Azure CycleCloud Workspace for Slurm (CCWS)](https://learn.microsoft.com/en-us/azure/cyclecloud/overview-ccws?view=cyclecloud-8)** — Slurm-based HPC clusters
- **[Azure Kubernetes Service (AKS)](https://learn.microsoft.com/en-us/azure/aks/what-is-aks)** — Kubernetes-based GPU clusters
- **[Azure Machine Learning](https://learn.microsoft.com/en-us/azure/machine-learning/?view=azureml-api-2)**

## Repository Structure

```
skills/slurm/                       # Domain knowledge skills (each subdir has SKILL.md)
infrastructure_validations/
  slurm/                            # Slurm validation tests (NCCL, GPU GEMM, thermal, NHC)
  aks/                              # AKS validation tests (NCCL, NHC, FIO)
infrastructure_references/
  azure_cyclecloud_workspace_for_slurm/  # CCWS deployment (Bicep, CLI, Event Grid)
  aks/                              # AKS deployment (deploy-aks.sh, operator configs)
examples/
  megatron-lm/                      # GPT3-175B training (Slurm + AKS)
  llm-foundry/                      # MPT training with LLM Foundry (Slurm + AKS)
  nemo-run/                         # NeMo-Run finetuning via Jupyter (Slurm)
  dgx_benchmarking/                 # DGX benchmark tuning for Azure (Slurm)
storage_references/
  aks/shared_storage/               # BlobFuse and AMLFS Helm charts for AKS
  slurm/squashed_images/            # Lustre striping and NVME staging for sqsh files
scheduling/aks/kueue/               # Kueue GPU queue management for AKS
utilities/aks/                      # Node labeler and torset labeler for AKS
tools/ai-infrastructure-mcp/        # MCP server for AI assistant ↔ cluster integration
```

## Skill-First Workflow

For any cluster operations, validation, or troubleshooting request:

1. Start with `skills/slurm/slurm_router/SKILL.md` to select the right skill(s).
2. Use exact commands, thresholds, and decision criteria from the selected skill files.
3. Do not provide generic HPC advice when a skill exists for that task.
4. If required inputs are missing (SKU, nodelist, cluster name, failing job details), ask for them.

### Skills Index

| Skill | Path | Purpose |
|-------|------|---------|
| Slurm Router | `skills/slurm/slurm_router/SKILL.md` | Intent → skill mapping (use first) |
| SKU Baseline | `skills/slurm/sku_performance_baseline/SKILL.md` | Expected values, warn/GHR thresholds |
| NCCL AllReduce | `skills/slurm/nccl_allreduce_test/SKILL.md` | Run and interpret NCCL bandwidth tests |
| NCCL Diagnosis | `skills/slurm/nccl_performance_diagnosis/SKILL.md` | Bisection algorithm, intra/inter-rack scoping |
| GPU Validation | `skills/slurm/node_gpu_validation/SKILL.md` | ubergemm GEMM benchmarks, fleet analysis |
| Thermal Test | `skills/slurm/thermal_stress_test/SKILL.md` | dcgmproftester stress test |
| IB Validation | `skills/slurm/ib_link_validation/SKILL.md` | Port state, pkeys, error counters |
| Rack Topology | `skills/slurm/rack_topology/SKILL.md` | MNNVL domains, ClusterUUID discovery |
| Outlier Detection | `skills/slurm/cluster_outlier_detection/SKILL.md` | Z-score, MAD, fleet-wide statistical analysis |
| Azure GHR | `skills/slurm/azure_node_health_report/SKILL.md` | Guest Health Report filing, impact categories |
| Node Lifecycle | `skills/slurm/node_drain_and_replace/SKILL.md` | Drain/undrain/reboot decision tree |

## SKU Quick Reference

| SKU | GPUs/node | IB ports | GPU GEMM target | NCCL intra-rack | NCCL inter-rack | Rack size |
|-----|-----------|----------|-----------------|-----------------|-----------------|-----------|
| GB300 (NDv6) | 4 | 4 × 400 Gb/s | ~1,850 TFlops | ~937 GB/s | ~200 GB/s | 18 nodes (MNNVL) |
| H100 (NDv5)  | 8 | 8 × 400 Gb/s | ~769 GFlops   | ~450 GB/s | —         | No MNNVL  |

See `skills/slurm/sku_performance_baseline/SKILL.md` for warn/GHR thresholds.

## Test Scripts

Each test has its own README with exact commands.

### Slurm Tests

| Test | Script | What it does |
|------|--------|--------------|
| NCCL AllReduce | `infrastructure_validations/slurm/NCCL/nccl_test.sh` | Multi-node all_reduce_perf with per-SKU configs (hopper, graceblackwell) |
| GPU GEMM | `infrastructure_validations/slurm/gpu_test/gpu_test.slurm` | Per-GPU ubergemm benchmark, outputs GFlops CSV |
| Thermal | `infrastructure_validations/slurm/thermal_test/thermal_test.slurm` | dcgmproftester stress, binary pass/fail |
| NHC | `infrastructure_validations/slurm/NHC/nhc.slurm` | Node health checks; run → reboot failures → rerun → drain |

### AKS Tests

| Test | Chart | Notes |
|------|-------|-------|
| NCCL AllReduce | `infrastructure_validations/aks/NCCL/helm/nccl-test` | Helm chart, configurable nodes/GPUs |
| NHC | `infrastructure_validations/aks/NHC/` | Containerized node health checks |
| FIO | `infrastructure_validations/aks/fio/helm/fio` | Storage I/O benchmarks (BlobFuse, AMLFS) |