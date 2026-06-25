---
layout: post
title: "Generating Grafana Dashboards Using LLMs"
date: 2025-10-01
categories: [ai, llm]
tags: [observability, grafana, prometheus, kubernetes, k3d]
source_code: "https://github.com/walterdeane/blog-drafts/tree/main/blogs/003_generating_grafana_dashboards_using_LLMs"
---

*Full article coming soon. In the meantime, the complete lab environment with source code is available on GitHub.*

This post walks through using an LLM to generate Grafana dashboards for a Kubernetes-based monitoring stack. The lab environment spins up a local K3d cluster with kube-prometheus-stack, a Go demo app exporting Prometheus metrics, and auto-loads generated Grafana dashboards.

**Source code and setup instructions:** [github.com/walterdeane/blog-drafts — post 003](https://github.com/walterdeane/blog-drafts/tree/main/blogs/003_generating_grafana_dashboards_using_LLMs)

## What's in the lab

- Local K3d cluster (Kubernetes in Docker)
- kube-prometheus-stack (Prometheus + Grafana)
- Go demo app with custom Prometheus metrics
- LLM-generated dashboard JSON loaded automatically via ConfigMap

## Prerequisites

- Docker Desktop with 16GB+ RAM
- 10GB+ free disk space

See the [GitHub repo](https://github.com/walterdeane/blog-drafts/tree/main/blogs/003_generating_grafana_dashboards_using_LLMs) for full setup instructions.
