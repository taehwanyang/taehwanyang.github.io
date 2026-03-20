---
title: "eBPF로 Pod Blocker 구현"
date: 2026-03-20
---

## 프로젝트 소개
  - 이 프로젝트는 eBPF로 IP 패킷을 조사해서 특정 Limit을 초과한 파드의 트래픽을 차단하는 Pod Blocker라는 프로그램을 구현합니다.

## connection_counter.c
### 동작 흐름

패킷이 들어오면:
  1. IPv4/TCP인지 확인
  2. SYN && !ACK 인지 확인
  3. dst_ip가 watch_dst_ips에 있는지 확인
  4. config_map에서 윈도우/임계치 읽음
  5. (dst_ip, src_ip) 상태 조회
  6. 윈도우가 지났으면 reset
  7. 아니면 count 증가
  8. count가 limit 초과면
  9. drop_events에 이벤트 기록
  10. TC_ACT_SHOT 반환
  11. 아니면 TC_ACT_OK

### XDP가 아니라 TC인 이유
  1. XDP (eth0에 attach) 
    XDP는 네트워크 스택 이전 네트워크 인터페이스에서 바로 패킷을 가로챈다.
  2. tc (Traffic Control)
    Pod의 네트워크 스택에 위치(가장 가깝다.)

  K3S의 경우
    Flannel(VXLAN)
      [Outer IP: Node → Node]
        [UDP: VXLAN]
          [Inner IP: Pod → Pod]
    VXLAN은 overlay network인데 Pod IP 패킷은 그대로 두고 Node IP 헤더로 감싸는 encapsulation
    XDP의 경우 Pod가 다른 노드의 Pod로 패킷을 전달하면 노드 IP만 보고 Pod IP는 직접 파싱해야 한다.
    tc의 경우
      incoming VXLAN: NIC -> XDP -> (VXLAN decap) -> tc ingress -> Pod
      outgoing VXLAN: Pod -> tc egress -> (VXLAN encapsulation) -> NIC
      두 경우 모두 Pod IP가 보인다.