---
title: "eBPF로 Pod Blocker 구현"
date: 2026-03-20
---

## 프로젝트 소개
  - eBPF는 리눅스 시스템프로그래밍 분야에서 각광받고 있는 기술로 커널을 건드리지 않고 Observability, Security, Performance 등에서 다양한 기능을 구현할 수 있다.
  - 이 프로젝트는 eBPF로 TCP SYN 패킷을 카운트하여 제한 조건을 초과한 파드 사이의 트래픽을 차단하는 Pod Blocker라는 프로그램을 구현한다.
  - auth-attacker는 keep-alive를 끈 채 대규모 HTTP 요청을 보냅니다. 이를 통해 TCP SYN flooding 공격을 비슷하게 구현할 수 있다.
  - 커널모드에서 동작하는 eBPF 프로그램은 (src-ip, dst-ip)의 SYN 요청 개수를 카운트하고 조건에 따라 패킷을 드롭한다. 
    - count_conn_and_drop.c
  - 유저모드에서 동작하는 Go 프로그램은 패킷 드롭을 트리거하는 제한 조건을 커널모드로 전달하고 tc hook을 보호하려는 파드의 호스트 veth 인터페이스에 추가한다.
    - create_tc_hook_and_show_drop_log.go
  - 호스트에서 eBPF 프로그램을 실행해도 되지만 일반적으로 쿠버네티스 클러스터에서는 DaemonSet으로 실행합니다. 이러한 방식은 클라우드 쿠버네티스에도 유효한 전략으로 AKS나 EKS 모두 지원합니다. cilium을 비롯해 Datadog의 에이전트, Falco, Pixie도 모두 DaemonSet으로 실행됩니다.

## 깃헙 레포지토리
  - [eBPF Pod Blocker](https://github.com/taehwanyang/ebpf-pod-blocker)

## 개발 스택
  - C : libbpf 
  - Go : ebpf-go
  - Java : spring security

## 개발환경 구축
  - Mac을 사용하고 있기 때문에 lima vm을 쓰기로 결정
  - ubuntu 환경에 k3s를 설치
  - 자세한 구축방법은 깃헙 레포지토리 README.me에 있음.

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

## 쿠버네티스 네트워크 구성
  - 쿠버네티스의 네트워크
    - 쿠버네티스에 파드가 생성되면 새로운 namespace(리눅스 커널의 네임스페이스, 리소스에 대한 VIEW를 제공, 예를 들어 어떤 네트워크 인터페이스가 해당 네임스페이스에서만 보인다 등)와 cgroup(CPU나 메모리 같은 자원을 격리한다.)를 생성한다.
    - 호스트와 파드 사이을 잇는 veth pair(veth는 virtual ethernet)가 생성되는데 하나는 host-side veth로 호스트 네임스페이스에 생기고 다른 하나는 pod-side veth로 파드 네임스페이스에 생긴다.
    - 개발환경인 k3s에는 노드 간 통신에서는 VXLAN(overlay network, 파드 IP를 그대로 가친채 노드 IP 헤더로 감싸는 encapsulation)으로, 같은 노드 내의 파드 간 통신에는 bridge로 구성된다.
    - 호스트에서는 host-side veth는 호스트에서만 보이고 pod-side veth는 파드 내부에서만 보인다.
    - auth-attacker 파드에서 authorization server 파드로 요청을 보내면 패킷은 아래와 같이 흐른다.
    - pod-side veth (auth-attacker namespace) -> host-side veth of auth-attacker (host) -> bridge (host) -> host-side veth of authorization-server (host) -> pod-side veth (authorization-server namespace)
    - bridge는 virtual L2 스위치로 노드 내 파드들을 묶는 역할을 한다. 
```bash
ip link
3: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether 1a:fc:43:c1:9c:b4 brd ff:ff:ff:ff:ff:ff
4: cni0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 4a:68:33:43:05:84 brd ff:ff:ff:ff:ff:ff
```

## eBPF DaemonSet 구성
  - eBPF 프로그램을 파드에서 실행하기 위해 호스트의 리소스에 접근해야 할 필요가 있다. 
  - hostNetwork: true
    - attach 대상 인터페이스가 host namespace에 있기 때문에 필요하다.
  - hostPath /sys/fs/bpf
    - pinned BFP maps, pinned BPF programs 저장
  - hostPath /sys/kernel/debug
    - bpf_printk 출력 확인
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: pod-blocker
  namespace: pod-blocker-system
spec:
  selector:
    matchLabels:
      app: pod-blocker
  template:
    metadata:
      labels:
        app: pod-blocker
    spec:
      serviceAccountName: pod-blocker
      hostNetwork: true
      hostPID: true
      nodeSelector:
        kubernetes.io/os: linux
      tolerations:
        - operator: Exists
      containers:
        - name: pod-blocker
          image: ythwork/pod-blocker:0.0.1
          imagePullPolicy: IfNotPresent
          securityContext:
            privileged: true
          env:
            - name: MY_NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: TARGET_NAMESPACE
              value: auth
            - name: LABEL_SELECTOR
              value: app=authorization-server
          volumeMounts:
            - name: bpffs
              mountPath: /sys/fs/bpf
            - name: debugfs
              mountPath: /sys/kernel/debug
            - name: lib-modules
              mountPath: /lib/modules
              readOnly: true
            - name: host-proc
              mountPath: /host/proc
              readOnly: true
      volumes:
        - name: bpffs
          hostPath:
            path: /sys/fs/bpf
            type: DirectoryOrCreate
        - name: debugfs
          hostPath:
            path: /sys/kernel/debug
            type: DirectoryOrCreate
        - name: lib-modules
          hostPath:
            path: /lib/modules
        - name: host-proc
          hostPath:
            path: /proc
```

## 구현 상세
### auth-attacker
  - authorization server로 HTTP 요청을 보내는데 keep alive를 비활성화한다.
  - 이렇게 하면 매 요청마다 커넥션을 다시 맺기 때문에 SYN flooding처럼 L4 레이어에 대한 공격을 모의할 수 있다.

```go
func main() {
  // ...
  transport := &http.Transport{
		DisableKeepAlives: true,
	}

	client := &http.Client{
		Timeout:   5 * time.Second,
		Transport: transport,
	}

  var wg sync.WaitGroup
	for i := 0; i < workerCount; i++ {
		wg.Add(1)
		go worker(&wg, client, jobs, &success, &failure)
	}
  // ...
}

func worker(
  wg *sync.WaitGroup,
	client *http.Client,
	jobs <-chan int,
	success *int64,
	failure *int64,
) {
  for jobID := range jobs {
		// ...
		err := sendRequest(client)
    // ...
  }
}

func sendRequest(client *http.Client) error {
  // ...  
	req, err := http.NewRequest(http.MethodPost, tokenURL, bytes.NewBufferString(form.Encode()))
  // ...
	resp, err := client.Do(req)
	// ...
}
```

### eBPF 커널모드 
  - 리눅스 커널에서 패킷 경로
    - NIC -> (XDP 위치) -> skb(패킷 객체) 생성 -> (tc ingress, 커널 네트워크 스택 입구) -> Ethernet(L2) -> IP(L3) -> TCP/UDP(L4) -> APPLICATION(L7) -> TCP/UDP(L4) -> IP(L3) -> Ethernet(L2) -> (tc egress, 커널 네트워크 스택 출구) -> NIC
  - tc
    - skb(패킷 객체)가 생성되는 시점을 커널 네트워크 스택의 시작점으로 보면 tc ingress는 커널 네트워크 스택의 입구라고 할 수 있고 
    - 패킷이 L2, L3, L4를 거쳐 애플리케이션 레이어까지 올라갔다가 다시 L4, L3, L2를 타고 내려온 후 거치는 tc egress는 커널 네트워크 스택의 출구라고 할 수 있다.
    - eBPF로 tc hook을 tc ingress로 특정 인터페이스에 걸면 패킷이 네트워크 스택 입구(Ethernet 진입 전)에서 원하는 로직을 실행할 수 있다.
    - isovalent는 cillium을 구현할 때 XDP나 tc ingress를 이용해 구현했으므로 kubernetes가 겪을 수 있는 문제인 SNAT Port 고갈이나 conntrack 테이블 메모리 이슈를 해결할 수 있었다. 
    - 패킷이 들어올 때 로직을 트리거하려면 특정 인터페이스에 clsact qdisc(queueing discipline)를 붙이고 여기에 tc ingress로 필터를 추가
    - 패킷이 나갈 때 로직을 트리거하려면 특정 인터페이스에 clsact qdisc(queueing discipline)를 붙이고 여기에 tc egress로 필터를 추가
  - 공격자 파드가 authorization server 파드로 대규모 커넥션 요청을 하면 이는 DDoS에 해당하므로 차단해야 하는 상황
  - 보호하고자 하는 파드와 최대 요청 가능 횟수는 eBPF 유저모드에서 전달받는다.
  - 참고로, 파드에서 오는 패킷이 아니라 노드 밖에서 오는 패킷이라면 XDP(eXpress Data Path)를 고려해볼 수 있는데 XDP는 패킷이 메모리에 올라가기 전이므로 속도가 매우 빠르다.
  - 하지만, tc는 커널 표준 패킷인 skb를 사용하므로 NetworkPolicy, ServiceLB, NAT 등 복잡한 기능을 구현하기 쉽지만 XDP는 NIC 드라이버에 의존적이며 구현 복잡도가 높고 디버깅이 어려워 특정 경우에만 사용할 수 있다.

```c
// target_ip가 보호하고자 하는 파드의 IP이다.
struct ip_pair_key {
    __u32 target_ip;
    __u32 src_ip;
};

// window_ns는 요청을 카운트하는 시간
// max_count는 최대 가능 요청 횟수
// 예를 들어, src 파드에서 target 파드로 1분에 100번 커넥션 요청이 들어오면 패킷을 드롭한다는 정책이 된다.
struct rl_config {
    __u64 window_ns;
    __u64 max_count;
    __u32 pad; 
};

// 설정을 저장할 자료구조
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 1);
    __type(key, __u32);
    __type(value, struct rl_config);
} config_map SEC(".maps");

// 유저모드에서 전달받은 보호할 파드 목록
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 4096);
    __type(key, __u32);
    __type(value, __u8);
} watch_dst_ips SEC(".maps");
```

  - 패킷이 들어올 때마다 (src-ip, dst-ip)를 기준으로 요청 횟수를 저장할 구조체와 패킷 드롭 정책이 적용되었을 때 이 이벤트를 유저모드로 전달할 자료구조가 필요하다.

```c
struct rl_state {
    struct bpf_spin_lock lock;
    __u64 window_start_ns;
    __u32 count;
    __u32 pad;
};

struct drop_event {
    __u64 ts_ns;
    __u32 target_ip;
    __u32 src_ip;
    __u32 count;
    __u32 max_count;
};

// (target_ip, src_ip)별 rate limit 상태 저장
// key = {target_ip, src_ip}
// value = rl_state
// 요청 횟수를 저장하기 위한 자료구조
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 65536);
    __type(key, struct ip_pair_key);
    __type(value, struct rl_state);
} target_src_states SEC(".maps");

// 패킷 드롭 이벤트를 유저모드로 전달하기 위한 링 버퍼
struct {
    __uint(type, BPF_MAP_TYPE_RINGBUF);
    __uint(max_entries, 1 << 24); // 16MiB
} drop_events SEC(".maps");
``` 

  - 자료구조를 정했으니 로직을 구현한다. 
    - (src->ip, dst->ip)를 키로 특정 시간동안 최대 가능 요청 횟수를 넘으면 이후의 모든 패킷을 드롭한다.
    - TC_ACT_OK : 패킷을 네트워크 스택에 넘긴다.
    - TC_ACT_SHOT : 패킷을 드롭한다.

```c
SEC("tc")
int count_syn_and_drop(struct __sk_buff *skb) 
{
    // ...
    //  패킷을 IP 헤더와 TCP 해더로 파싱
    if (parse_ipv4_tcp(skb, &ip, &tcp) < 0) {
        return TC_ACT_OK;
    }
    // ...
    // 요청 횟수를 기록 중인 맵을 가져온다
    state = bpf_map_lookup_elem(&target_src_states, &pair_key);
    // ...
    // 리밋을 넘지 않았다면 횟수에 1을 더한다.
    state->count += 1;
    // 요청 횟수가 최대 가능 요청 횟수를 초과하면 패킷을 드롭한다.
    if (state->count > cfg->max_count) {
        __u32 current_count = state->count;
        __u32 max_count = cfg->max_count;
        __u32 target_ip = pair_key.target_ip;
        __u32 src_ip = pair_key.src_ip;

        bpf_spin_unlock(&state->lock);

        bpf_printk("drop target=%x src=%x count=%u\n",
               target_ip, src_ip, current_count);

        emit_drop_event(now_ns, target_ip, src_ip, current_count, max_count);
        return TC_ACT_SHOT;
    }
    // ...    
    return TC_ACT_OK;
}
```

### eBPF 유저모드
  - 패킷을 관찰하다가 최대 요청 횟수를 넘기면 DROP 정책을 실행할 tc hook 위치를 정해야 한다.
  - eBPF 프로그램은 호스트에서 실행되므로 authorization server 파드 내에 있는 pod-side veth는 보이지 않는다.
  - 그러므로 authorization server의 host-side veth에 tc egress로 필터를 추가한다.
  - 공격자 파드 -> bridge(cni0) -> host-side veth -> (tc hook 위치, host-side veth에서는 패킷이 나가는 경우이므로 tc egress) -> pod-side veth
  - 먼저 정책(어떤 파드에 몇분 동안 커넥션 요청이 몇번 이상 들어오면 패킷을 드롭한다.)을 설정한다.

```go
type Agent struct {
	objs     count_conn_and_dropObjects
	tcClient *tc.Tc
	watchSet map[uint32]struct{}
	ifIndex  uint32
	ifName   string
	tcHandle uint32
}

func CreateTCHookAndShowDropLog(ctx context.Context) error {
  // ...
  // app=authorization-server 레이블을 가진 파드를 가져온다.
  pods := PodsByLabel()
  
  if err := agent.applyRateLimitConfig(Window, MaxCount); err != nil {
    // ...
  }
  // ...
  if err := agent.setWatchIPs(podIPs); err != nil {
    // ...
  }
  // ...
}
```

  - authorization server 파드의 host-side veth를 가져와야 하는데 아래 명령을 Go 코드로 구현하면 된다.

```bash
kubectl exec auth-test-authorization-server-7ddc9bcc8d-kqd87 -- cat /sys/class/net/eth0/iflink
12
ip link
# veth54c1b554가 authorization server 파드의 host-side veth이다.
12: veth54c1b554@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master cni0 state UP mode DEFAULT group default qlen 1000
    link/ether 42:33:3d:19:04:6d brd ff:ff:ff:ff:ff:ff link-netns cni-4f918ed8-3dcf-7039-ca38-3602426ed6cc
```
  - authorization server의 host-side veth를 가져온 후 clsact qdisc를 붙이고 tc hook을 추가한다.

```go
func CreateTCHookAndShowDropLog(ctx context.Context) error {
  // ...
  hostVeth, err := hostVethFromPod(pods)
  // ...
  iface, err := net.InterfaceByName(hostVeth)
  // ...
  tcClient, err := tc.Open(&tc.Config{})
  // ...
  if err := resetClsact(agent.tcClient, agent.ifIndex); err != nil {
    // ...
  }
  // ...
  if err := attachBPFProgram(
		agent.tcClient,
		agent.ifIndex,
		agent.objs.CountSynAndDrop.FD(),
		"count_syn_and_drop",
		agent.tcHandle,
	); err != nil {
    // ...
  }
  // ...
}

func attachBPFProgram(tcnl *tc.Tc, ifindex uint32, progFD int, progName string, handle uint32) error {
	fd := uint32(progFD)
	flags := uint32(0x1) // direct-action

	filter := tc.Object{
		Msg: tc.Msg{
			Family:  unix.AF_UNSPEC,
      // authorization server의 host-side veth
			Ifindex: ifindex,
			Handle:  handle,
      // tc hook을 host-side veth의 tc egress로 걸어야 한다.
			Parent:  core.BuildHandle(tc.HandleRoot, tc.HandleMinEgress),
			Info:    core.FilterInfo(0, unix.ETH_P_ALL),
		},
		Attribute: tc.Attribute{
			Kind: "bpf",
			BPF: &tc.Bpf{
				FD:    &fd,
				Name:  &progName,
				Flags: &flags,
			},
		},
	}
	return tcnl.Filter().Add(&filter)
}
```

  - 커널모드에서 패킷 드롭이 발생하면 이벤트를 가져와 로그로 출력한다.

```go
func CreateTCHookAndShowDropLog(ctx context.Context) error {
  // ...
  return runDropEventLoop(ctx, reader)
}

func runDropEventLoop(ctx context.Context, reader *ringbuf.Reader) error {
  for {
		record, err := reader.Read()
    // ...
    var evt dropEvent
		if err := binary.Read(bytes.NewReader(record.RawSample), binary.LittleEndian, &evt); err != nil {
      // ...
    }

    log.Printf(
			"[DROP] recv_time=%s src=%s dst=%s count=%d limit=%d kernel_ts_ns=%d",
			time.Now().Format(time.RFC3339),
			u32ToIP(evt.SrcIp),
			u32ToIP(evt.TargetIp),
			evt.Count,
			evt.MaxCount,
			evt.TsNs,
		)
  }
}
``` 
