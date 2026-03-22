---
title: "eBPF로 Pod Blocker 구현"
date: 2026-03-20
---

## 프로젝트 소개
  - eBPF는 리눅스 시스템프로그래밍 분야에서 각광받고 있는 기술로 커널을 건드리지 않고 Observability, Security, Performance 등에서 다양한 기능을 구현할 수 있습니다.
  - 이 프로젝트는 eBPF로 TCP SYN 패킷을 카운트하여 제한 조건을 초과한 파드 사이의 트래픽을 차단하는 Pod Blocker라는 프로그램을 구현합니다.
  - auth-attacker는 keep-alive를 끈 채 대규모 HTTP 요청을 보냅니다. 이를 통해 TCP SYN flooding 공격을 비슷하게 구현할 수 있습니다.
  - 커널모드에서 동작하는 eBPF 프로그램은 (src-ip, dst-ip)의 SYN 요청 개수를 카운트하고 조건에 따라 패킷을 드롭합니다. 
    - count_conn_and_drop.c
  - 유저모드에서 동작하는 Go 프로그램은 패킷 드롭을 트리거하는 제한 조건을 커널모드로 전달하고 tc hook을 보호하려는 파드의 호스트 veth 인터페이스에 추가합니다.
    - create_tc_hook_and_show_drop_log.go

## 개발 스택
  - C : libbpf 
  - Go : ebpf-go
  - Java : spring security

## 개발환경 구축
  - Mac을 사용하고 있기 때문에 lima vm을 쓰기로 결정
  - ubuntu 환경에 k3s를 설치
  - 자세한 구축방법은 깃헙 레포지토리 README.me에 있습니다.

## 시나리오
  - 보호하고자 하는 리소스
    - auth라는 namespace에 OAuth2를 구현한 Authorization Server(테스트의 편의를 위해 grant_type이 password로 구현, 현재 password 타입은 OAuth2에서 제외됐다.)와 Resource Server 파드가 각각 하나씩 작동 중
  - 공격자 파드
    - attack이라는 namespace에 auth-attacker 파드는 50개의 worker가 authorization server에 대규모 인증 요청을 보낸다.
  - NDR(Network Detection & Response)
    - eBPF 프로그램은 쿠버네티스가 아닌 호스트에서 실행된다. 
    - 커널모드에서는 유저모드에서 전달받은 설정을 기반으로 특정 파드(authorization server)로 향하는 커넥션 요청 패킷을 카운트하고 있다가 최대 가능 요청 수를 넘기면 이후의 커넥션 요청 패킷을 드롭한다. 
    - 이와 같이 이상행동을 찾아내고 대응하는 것을 보안 분야에서는 Detection & Response이라고 부른다. 대표적으로 EDR(Endpoint Detection & Response)이나 CDR(Cloud Detection & Response)
    - 유저모드에서는 보호하고자 하는 파드(authorization server) 및 최대 가능 요청 수 같은 설정을 커널모드에 전달하며 tc hook을 보호하고자 하는 파드의 host-side veth 인터페이스에 추가한다.




