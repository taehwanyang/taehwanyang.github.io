---
title: "eBPF로 HPA 구현해보기"
date: 2026-03-20
---

## 프로젝트 소개
  - 이 프로젝트는 eBPF를 통해 HPA(Horizontal Pod Autoscaler)를 간단하게 구현하는 PoC입니다.
  - metrics-server가 하는 일을 커널모드에서 eBPF로 구현합니다. 
  - 커널모드에서 수집한 metric을 기반으로 ebpf-go로 구현한 유저 모드 애플리케이션에서 파드 스케일링을 합니다. 
  - eBPF는 리눅스 시스템 프로그래밍 분야에서 각광받고 있는 기술로 이를 활용하면 클라우드 운영에서 Observability, Security 등에서 다양한 기능을 구현할 수 있습니다.
  - 이 프로젝트는 eBPF로 어떻게 개발을 해야 하는지 보여주는 간단한 데모입니다.
  - eBPF를 활용하려는 많은 개발자들에게 도움이 되었으면 좋겠습니다.