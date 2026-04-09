---
title: "Event-driven 아키텍처로 구현한 DDD 모듈러 모놀리스 애플리케이션"
date: 2026-04-08
---

## 프로젝트 소개
  - 리눅스 OS에는 많은 IO 모델이 있다.
    - 대표적으로 Multiprocessing, Multithreading, I/O Multiplexing
    - C 계열을 제외한 프로그래밍 언어의 경우 대부분 추상화된 IO를 제공
    - 자바로 http 서버를 개발한다면 Tomcat과 Netty에 대해서 알고 있을텐데 Tomcat은 일반적으로 Multithreading으로 요청을 처리한다. Netty는 EventLoop와 EventLoopGroup을 제공하는데 EventLoop가 리눅스의 경우 epoll로 구현된 I/O Multiplexing이고 EventLoopGroup은 Multithreading으로 각 스레드에서 I/O Multiplexing으로 IO를 처리한다.
    - Java의 Netty나 Python의 asyncio 같은 이벤트루프 기반의 비동기 프레임워크의 내부 동작이 리눅스에서는 어떻게 구현되는지 설명한다.
      - 이벤트루프는 리눅스에서는 epoll, 윈도우는 IOCP, macos에서는 kqueue로 구현된다.
    - 프로그램은 단순히 클라이언트의 응답을 그대로 전달하는 Echo server를 작성했다.
    - 소켓은 NonBlocking으로 설정했다. 실무에서 Blocking Socket을 사용하는 일은 지금까지 한번도 없었다. 
    - epoll 함수는 소켓 버퍼에 데이터가 있을 때 두가지 방식으로 작동하는데 이번 프로젝트에서는 Edge Trigger로 구현했다.
      - Edge Trigger
        - 데이터가 없다가 있을 때만, 즉 데이터가 새로 들어왔을 때만 epoll wakeup 발생
      - Level Trigger
        - 버퍼에 데이터가 남아있으면 계속 epoll wakeup이 발생

## 깃헙 레포지토리
  - [I/O Multiplexing Threadpool Echo Server](https://github.com/taehwanyang/io-multiplexing-threadpool-echo-server)

## 개발스택
  - 언어 : C 언어
  - 네트워크 프로그래밍 : Berkeley Socket
  - 멀티스레딩 프로그래밍 : pthread (POSIX Thread)
  - 컴파일러 : gcc

## 컴파일
```sh
gcc -O2 -Wall -Wextra -pthread echo_epoll_threadpool.c -o echo_server
```
 
 ## 실행
 ```sh
 ./echo_server
 ```

## 테스트
```sh
nc 127.0.0.1 8080
```

## I/O Multiplexing Threadpool
  - 일반적인 Multiprocessing이나 Multithreading 모델의 경우 한 프로세스나 스레드가 하나의 소켓(ESTABLISHED 상태로 클라이언트와 연결되어 있고 write와 send가 가능한 데이터 소켓)을 담당하는데 대부분 IO bound한 작업이 많다. 클라이언트의 수가 많이 늘어나면 프로세스와 스레드의 수가 비례하여 늘어나므로 서버에 큰 부담이 된다.
    - Multithreading이 Multiprocessing보다는 그나마 조금 나은데 이는 컨텍스트 스위칭과 CPU 안에 있는 L1, L2 캐시의 cache hit까지 언급해야 하므로 여기에서는 다루지 않겠다.
  - 이에 비해 I/O Multiplexing은 한 스레드 안에서 여러 소켓을 다룰 수 있다. 
    - 리눅스에서는 모든 것이 파일이라는 철학이 있는데 소켓도 파일이다. 소켓이 생성되면 FD(File Descriptor)를 가진다. 
    - 이 FD를 epoll에 등록하면 epoll은 잠들어 있다가(epoll_wait) 소켓 버퍼에 새로운 데이터가 들어오거나 읽을 수 있는 데이터가 남아있을 때만 깨어나(epoll wakeup) 데이터를 읽어 TCP Payload를 완성한 후 로직을 실행할 수 있다.(TCP는 경계가 없는 바이트스트림이므로 이 바이트를 취합해 Payload를 완성하는 것도 프로그래머의 몫이다.)
    - 한 스레드에서 많은 소켓을 다룰 수 있는 장점이 있지만 소켓에서 읽은 데이터를 실행하는 로직이 CPU bound한 작업이라면 epoll_wait로 진입하는 시점이 늦어져 여러 소켓들이 버퍼에 읽을 데이터가 있음에도 읽는 시점이 늦어질 수 있다는 단점이 있다. 
    - 그러므로 I/O Multiplexing을 싱글 스레드에서 운영한다면 반드시 IO bound한 작업만 실행해야 한다.
    - 이를 보완하기 위해서 스레드풀을 도입할 수 있다.
      - 코어 수만큼 스레드를 만들어 스레드풀을 만들고 스레드풀의 각 스레드는 I/O Multiplexing으로 데이터를 받아 처리하면 된다.
      - Netty의 EventLoop는 I/O Multiplexing을 추상화한다.(리눅스는 epoll, 윈도우는 IOCP, macos는 kqueue, 한가지 짚고 넘어갈 점은 IOCP는 I/O Multiplexing은 아니다. 다만 Netty나 asyncio는 select 게열 함수를 구현할 때 내부적으로 윈도우는 IOCP를 사용해 I/O multiplexing을 에뮬레이팅한다.)
      - Netty의 EventLoopGroup은 스레드풀을 추상화한다.
    - 예전에 게임 서버를 개발할 때는 데이터 입출력은 I/O Multiplexing Single Thread로 처리했고 게임 클라이언트로 부터 받은 데이터는 게임 로직을 처리하는 스레드풀로 넘겨 처리한 후 다시 I/O Multiplexing Single Thread로 넘겨 모든 클라이언트에 브로드캐스팅하도록 구현했다.

## 구현 상세
  - conn은 TCP 연결을 표현한다. 이 소켓 fd를 관리하는 worker 스레드 정보도 가지고 있다.
  - fd_node는 worker 스레드가 epoll에 등록하기 전의 소켓을 링크드리스트에 저장해두는게 이 리스트에 저장될 노드이다.
  - worker는 worker 스레드를 표현한다. 
    - epfd는 스레드가 관리하는 epoll 객체
    - notify_fd는 main 스레드에서 epoll_wait로 잠들어 있는 worker 스레드를 깨우는 역할을 한다.
    - queue_head와 queue_tail은 epoll 등록 대기 상태의 소켓을 담은 링크드리스트 구현
    - conns는 스레드가 가진 epoll 객체가 관리하는 소켓 배열

```c
struct conn {
    int fd;
    worker_t *owner;

    // pending write buffer
    char *wbuf;
    size_t wbuf_len;
    size_t wbuf_sent;
};

struct fd_node {
    int fd;
    fd_node_t *next;
};

struct worker {
    int id;
    int epfd;
    int notify_fd;
    pthread_t thread;

    pthread_mutex_t queue_lock;
    fd_node_t *queue_head;
    fd_node_t *queue_tail;

    conn_t *conns[MAX_FDS];
};
```

  - 소켓은 Notblocking으로 설정한다.

```c
static int set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    if (flags == -1) return -1;
    if (fcntl(fd, F_SETFL, flags | O_NONBLOCK) == -1) return -1;
    return 0;
}
```

  - EPOLL_CTL_MOD는 fd를 감시하는 방석(EPOLLIN, EPOLLOUT 등)을 바꾸는 작업을 의미

```c
static int mod_epoll_events(worker_t *w, int fd, uint32_t events) {
    struct epoll_event ev;
    memset(&ev, 0, sizeof(ev));
    ev.events = events;
    ev.data.fd = fd;
    return epoll_ctl(w->epfd, EPOLL_CTL_MOD, fd, &ev);
}
```

  - EPOLL_CTL_ADD는 새로운 fd를 등록

```c
static int add_epoll_fd(worker_t *w, int fd, uint32_t events) {
    struct epoll_event ev;
    memset(&ev, 0, sizeof(ev));
    ev.events = events;
    ev.data.fd = fd;
    return epoll_ctl(w->epfd, EPOLL_CTL_ADD, fd, &ev);
}
```

  - 읽은 데이터를 즉시 클라이언트로 보낸다. 다 못보내면 write buffer에 저장한다.
  - 한가지 중요한 점은 버클리 소켓 API의 send()는 항상 다 보내주지 않는다. 이를 관리하는 것도 프로그래머의 몫이다.

```c
static int handle_payload(conn_t *c, const char *buf, size_t len) {
    if (c->wbuf_len == 0) {
        // ...
        while (sent < len) {
            ssize_t n = send(c->fd, buf + sent, len - sent, 0);
            if (n > 0) {
                sent += (size_t)n;
                continue;
            }
            // ...
        }
    // ...
}
```

  - EPOLLIN 이벤트가 왔을 때 socket 데이터를 끝까지 읽고(EAGAIN까지), 읽은 데이터를 처리(handle_payload)한다.

```c
static void handle_readable(worker_t *w, conn_t *c) {
    // ...
    for (;;) {
        ssize_t n = recv(c->fd, buf, sizeof(buf), 0);
        if (n > 0) {
            int rc = handle_payload(c, buf, (size_t)n);
            // ...
            // epoll 이벤트 재설정
            // EPOLLIN 읽기 가능 상태(readable) 감시
            // EPOLLET Edge Trigger
            // EPOLLRDHUP peer가 FIN 보냄을 감지(half-close)
            uint32_t events = EPOLLIN | EPOLLET | EPOLLRDHUP;
            // ...

            if (mod_epoll_events(w, c->fd, events) == -1) {
                // ...
            }
            continue;
        }
        if (n == 0) {
            // peer closed
            // ...
        }
        if (errno == EAGAIN || errno == EWOULDBLOCK) {
            // ET mode: drained all readable data
            break;
        }
        close_connection(w, c->fd);
        return;
    }
}
```

  - EPOLLOUT 이벤트가 왔을 때 남아있는 write buffer를 flush한다.

```c
static void handle_writable(worker_t *w, conn_t *c) {
    // ...
    int rc = flush_write_buffer(c);
    // ...
    uint32_t events = EPOLLIN | EPOLLET | EPOLLRDHUP;
    if (c->wbuf_len > 0) {
        events |= EPOLLOUT;
    }
    if (mod_epoll_events(w, c->fd, events) == -1) {
        // ...
    }
}
```

  - main thread가 accpet한 소켓을 라운드 로빈으로 특정 worker thread의 큐에 넣고 그 worker thread를 깨운다.
    - worker thread는 epoll_wait로 잠들어 있어서 깨워야 함.

```c
static void enqueue_fd(worker_t *w, int fd) {
    fd_node_t *node = (fd_node_t *)malloc(sizeof(fd_node_t));
    // ...
    node->fd = fd;
    node->next = NULL;
    // 링크드 리스트에 Insert 연산
    pthread_mutex_lock(&w->queue_lock);
    if (w->queue_tail) {
        w->queue_tail->next = node;
        w->queue_tail = node;
    } else {
        w->queue_head = w->queue_tail = node;
    }
    pthread_mutex_unlock(&w->queue_lock);
    // worker thread를 깨운다.
    uint64_t one = 1;
    if (write(w->notify_fd, &one, sizeof(one)) == -1) {
        // best effort
    }
}
``` 

  - worker thread의 큐에 쌓인 fd를 전부 꺼내서 conn 객체로 만들고 epoll에 등록한다.

```c
static void drain_new_connections(worker_t *w) {
    for (;;) {
        // 링크드리스트에서 아직 epoll에 등록되지 않은 소켓을 하나 가져온다.
        fd_node_t *node = NULL;
        pthread_mutex_lock(&w->queue_lock);
        if (w->queue_head) {
            node = w->queue_head;
            w->queue_head = node->next;
            if (!w->queue_head) {
                w->queue_tail = NULL;
            }
        }
        pthread_mutex_unlock(&w->queue_lock);
        // ...
        // calloc은 malloc과 달리 메모리를 할당하면서 동시에 0으로 초기화한다.
        // conn 객체 생성
        conn_t *c = (conn_t *)calloc(1, sizeof(conn_t));
        // ...
        c->fd = fd;
        c->owner = w;
        w->conns[fd] = c;
        // epoll 등록
        if (add_epoll_fd(w, fd, EPOLLIN | EPOLLET | EPOLLRDHUP) == -1) {
            // ...
        }
        // ...
    }
}
```

  - worker thread는 epoll 기반으로 이벤트를 기다렸다가 fd별로 read/write/close 처리를 수행한다.

```c
static void *worker_main(void *arg) {
    // ...
    while (!stop_flag) {
        // 이벤트가 발생할 때까지 잠들어 있는다.
        int n = epoll_wait(w->epfd, events, MAX_EVENTS, 1000);
        // ...
        for (int i = 0; i < n; i++) {
            int fd = events[i].data.fd;
            uint32_t ev = events[i].events;
            // main thread가 새로운 커넥션이 있다고 알림
            if (fd == w->notify_fd) {
                // ...
                drain_new_connections(w);
                continue;
            }
            // ...
            // readable 이벤트 처리
            if (ev & EPOLLIN) {
                handle_readable(w, c);
                // ...
            }
            // writable 이벤트 처리
            if (ev & EPOLLOUT) {
                // ...
                handle_writable(w, c);
            }
        }
    }
    // ...
}
```

  - worker thread 초기화

```c
static void init_worker(worker_t *w, int id) {
    // ...
    // worker 전용 이벤트 감시 객체 생성
    w->epfd = epoll_create1(0);
    // ...
    // worker thread 생성
    if (pthread_create(&w->thread, NULL, worker_main, w) != 0) {
        // ...
    }
}
```

  - 리스닝 소켓을 생성 및 설정하고(listen() 함수 호출) 클라이언트 연결을 계속 받을 수 있도록 accept() 함수를 게속 호출한다.

```c
static int create_server_socket(uint16_t port) {
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    // ...
    int opt = 1;
    if (setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt)) == -1) {
        // ...
    }
    // ...
    if (set_nonblocking(listen_fd) == -1) {
        // ...
    }
    // ...
    if (bind(listen_fd, (struct sockaddr *)&addr, sizeof(addr)) == -1) {
        // ...
    }
    if (listen(listen_fd, BACKLOG) == -1) {
        // ...
    }
    // ...
}

static void accept_loop(int listen_fd) {
    while (!stop_flag) {
        // ...
        int client_fd = accept(listen_fd, (struct sockaddr *)&client_addr, &client_len);
        // ...
        if (set_nonblocking(client_fd) == -1) {
            // ...
        }
        // ...
        // round robin 방식으로 할당
        worker_t *w = &workers[next_worker_idx];
        next_worker_idx = (next_worker_idx + 1) % worker_count;
        enqueue_fd(w, client_fd);
    }
}
```

  - main thread는 worker thread를 생성하고 서버 소켓을 열고 accept 루프를 돌리며 종료 시 정리를 수행한다.

```c
int main(int argc, char **argv) {
    // ...
    // 코어 수를 가져온다.
    long cores = sysconf(_SC_NPROCESSORS_ONLN);
    // ...
    worker_count = (int)cores;
    // ...
    // worker 초기화
    for (int i = 0; i < worker_count; i++) {
        init_worker(&workers[i], i);
    }
    // 서버 소켓 생성
    int listen_fd = create_server_socket(port);
    // ...
    // accpet 루프 시작
    accept_loop(listen_fd);
    // ...
}
```
