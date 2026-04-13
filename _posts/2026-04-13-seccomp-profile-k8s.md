---
title: "k8s에 Spring Boot Application을 위한 seccomp profile 적용"
date: 2026-04-13
---

## seccomp란?
  - seccomp (secure computing mode)
  - Linux 커널 기능으로, 프로세스가 사용할 수 있는 시스템 콜(syscall)을 제한하는 보안 메커니즘
  - 프로세스가 커널에게 요청할 수 있는 행동을 화이트리스트로 제한하는 기능

## 컨테이너 보안을 위한 seccomp
  - Docker/Kubernetes에서는 보통 
    - 기본 seccomp 프로파일인 RuntimeDefault을 사용하거나
    - strace로 startup부터 app runtime까지 syscall을 추적하여 커스텀 seccomp 프로파일을 만든다.
  - 이를 통해 공격자가 shell을 탈취해도 할 수 있는 행동이 제한되어 공격 표면을 감소하고 컨테이너 보안을 강화할 수 있다.
  - 단점은 서비스별로 syscall이 다르므로 seccomp 프로파일에 대한 반복/지속적인 튜닝이 필요하다.
    - syscall이 누락되면 앱이 바로 죽는다.

## 프로젝트 소개
  - Spring Boot 애플리케이션을 위한 커스텀 seccomp 프로파일을 작성하고 Deployment에 적용해본다.

## 🔥 seccomp profile 생성
  - kubernetes의 Deployment에 seccomp 프로파일을 적용

## 먼저 RuntimeDefault 적용
  - Authorization Server Deployment에 RuntimeDefault를 적용해본다.
  - 그럼 OCI 런타임의 기본 seccomp 프로파일이 적용된다.

```yaml
        containers:
          # ...
          securityContext:
            allowPrivilegeEscalation: false
            seccompProfile:
              type: RuntimeDefault
```

### strace로 startup부터 app runtime의 syscall을 추적하기 위해 테스트용 Dockerfile 만들기
  - -ff
    - 스레드/프로세스별로 파일 분리
    - /tmp/trace.<pid> 형태로 생김

  - -s 256
    - 문자열 더 길게 기록
  - -tt
    - 타임스탬프 포함
  -o /tmp/trace
    - 로그 파일 prefix

```sh
# ...
RUN apt-get update \
    && apt-get install -y strace \
    && rm -rf /var/lib/apt/lists/*
# ...
ENTRYPOINT ["strace", "-ff", "-s", "256", "-tt", "-o", "/tmp/trace", "java", "-jar", "/app/app.jar"]
```

### Resource Server startup ~ app runtime syscall 파일 만들기

```sh
kubectl exec -it auth-test-resource-server-767b699969-dm7gl -n auth -- sh -c 'tar czf /tmp/trace.tar.gz /tmp/trace*'
kubectl cp auth/auth-test-resource-server-767b699969-dm7gl:/tmp/trace.tar.gz ./trace.tar.gz

tar xzf trace.tar.gz
cd tmp

cat trace* | awk '{print $2}' | sed 's/(.*//' | sort -u
```

### Spring Boot Application seccomp profile 완성본

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "defaultErrnoRet": 1,
  "architectures": [
    "SCMP_ARCH_AARCH64"
  ],
  "syscalls": [
    {
      "names": [
        "accept",
        "bind",
        "brk",
        "clock_getres",
        "clock_nanosleep",
        "close",
        "close_range",
        "connect",
        "epoll_create1",
        "epoll_ctl",
        "epoll_pwait",
        "eventfd2",
        "execve",
        "exit",
        "faccessat",
        "fchdir",
        "fcntl",
        "flock",
        "fstat",
        "fstatfs",
        "ftruncate",
        "futex",
        "getcwd",
        "getdents64",
        "geteuid",
        "getpid",
        "getrandom",
        "getrusage",
        "getsockname",
        "getsockopt",
        "gettid",
        "getuid",
        "ioctl",
        "listen",
        "lseek",
        "madvise",
        "mkdirat",
        "mmap",
        "mprotect",
        "munmap",
        "newfstatat",
        "openat",
        "openat2",
        "ppoll",
        "prctl",
        "pread64",
        "prlimit64",
        "read",
        "readlinkat",
        "recvfrom",
        "rseq",
        "rt_sigaction",
        "rt_sigprocmask",
        "rt_sigreturn",
        "sched_getaffinity",
        "sched_yield",
        "sendmmsg",
        "set_robust_list",
        "set_tid_address",
        "setsockopt",
        "shutdown",
        "socket",
        "socketpair",
        "statx",
        "sysinfo",
        "uname",
        "write"
      ],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "names": ["clone"],
      "action": "SCMP_ACT_ALLOW",
      "args": [
        {
          "index": 0,
          "value": 2114060288,
          "op": "SCMP_CMP_MASKED_EQ"
        }
      ]
    },
    {
      "names": ["clone3"],
      "action": "SCMP_ACT_ERRNO",
      "errnoRet": 38
    }
  ]
}
```

## seccomp profile을 seccomp profile 폴더로 복사

```sh
cp profiles/springboot-java-v2.1.json /var/lib/kubelet/seccomp/profiles/springboot-java-v2.1.json
```

### Resource Server Deployment에 반영

```yaml
        containers:
          # ...
            securityContext:
              allowPrivilegeEscalation: false
              seccompProfile:
                type: Localhost
                localhostProfile: profiles/springboot-java-v2.1.json
```