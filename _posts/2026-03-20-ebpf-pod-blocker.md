---
title: "eBPF로 Pod Blocker 구현"
date: 2026-03-20
---

## 프로젝트 소개
  - eBPF는 리눅스 시스템 프로그래밍 분야에서 각광받고 있는 기술로 이를 활용하면 클라우드 운영에서 Observability, Security 등에서 다양한 기능을 구현할 수 있습니다.
  - 이 프로젝트는 eBPF로 TCP SYN 패킷을 카운트하여 제한 조건을 초과한 파드 사이의 트래픽을 차단하는 Pod Blocker라는 프로그램을 구현합니다.
  - auth-attacker는 keep-alive를 끈 채 대규모 HTTP 요청을 보냅니다. 이를 통해 TCP SYN flooding 공격을 비슷하게 구현할 수 있습니다.
  - 커널모드에서 동작하는 eBPF 프로그램은 (src-ip, dst-ip)의 SYN 요청 개수를 카운트하고 조건에 따라 패킷을 드롭합니다. 
    - count_conn_and_drop.c
  - 유저모드에서 동작하는 Go 프로그램은 패킷 드롭을 트리거하는 제한 조건을 커널모드로 전달하고 tc hook을 bridge 인터페이스에 추가합니다.
    - create_tc_hook_and_show_drop_log.go


