---
title: "eBPF로 Pod Blocker 구현"
date: 2026-03-20
---

## 프로젝트 소개
  - 이 프로젝트는 eBPF로 IP 패킷을 조사해서 특정 Limit을 초과한 파드의 트래픽을 차단하는 Pod Blocker라는 프로그램을 구현합니다.