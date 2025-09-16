---
layout: post
title: "Containerizing Apache NiFi — the pragmatic way"
date: 2025-09-16 10:00:00 +0530
tags: [NiFi, Containers, DevOps]
categories: [Architecture]
excerpt: "Step-by-step case study covering Docker, systemd, and production configs used to run NiFi at scale."
---

This is the **intro paragraph** — write a short TL;DR.

## Background

Explain why you containerized NiFi and what goals you had.

## Commands & examples

```bash
docker build -t my-nifi:1.0 .
docker run --name nifi -p 8080:8080 my-nifi:1.0
