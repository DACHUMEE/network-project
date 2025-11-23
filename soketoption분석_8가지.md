# 프로그램 코드와 설명

## 1. boradcast_recv

```cpp
// broadcast_recv.cpp
#define _WINSOCK_DEPRECATED_NO_WARNINGS
#include <winsock2.h>
#include <ws2tcpip.h>
#include <stdio.h>

#pragma comment(lib, "ws2_32.lib")

int main(int argc, char *argv[]) {
    WSADATA wsa;
    SOCKET s;
    struct sockaddr_in addr, from;
    char buf[1024];
    int port = 50001;

    if (argc >= 2) port = atoi(argv[1]);

    WSAStartup(MAKEWORD(2,2), &wsa);

    s = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
    if (s == INVALID_SOCKET) {
        printf("socket error=%d\n", WSAGetLastError());
        return 1;
    }

    addr.sin_family      = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    addr.sin_port        = htons(port);

    if (bind(s, (SOCKADDR*)&addr, sizeof(addr)) == SOCKET_ERROR) {
        printf("bind error=%d\n", WSAGetLastError());
        closesocket(s);
        WSACleanup();
        return 1;
    }

    printf("[RECV] Listening on UDP port %d ...\n", port);

    while (1) {
        int fromlen = sizeof(from);
        int n = recvfrom(s, buf, sizeof(buf)-1, 0,
                         (SOCKADDR*)&from, &fromlen);
        if (n == SOCKET_ERROR) {
            printf("recvfrom error=%d\n", WSAGetLastError());
            break;
        }
        buf[n] = '\0';
        printf("[RECV] From %s:%d : %s",
               inet_ntoa(from.sin_addr), ntohs(from.sin_port), buf);
    }

    closesocket(s);
    WSACleanup();
    return 0;
}

```
50001번으로 들어오는 브로드캐스트 메시지를 받아서 출력하는 프로그램이다.



## 2. boradcast_send

```cpp
// broadcast_send.cpp
#define _WINSOCK_DEPRECATED_NO_WARNINGS
#include <winsock2.h>
#include <ws2tcpip.h>
#include <stdio.h>

//#pragma comment(lib, "ws2_32.lib")

int main(int argc, char *argv[]) {
    WSADATA wsa;
    SOCKET s;
    struct sockaddr_in addr;
    int port = 50001;
    int enableBcast = 0;
    const char *msg = "Hello, broadcast!\n";

    if (argc >= 2) enableBcast = atoi(argv[1]); // 0 or 1
    if (argc >= 3) port        = atoi(argv[2]);

    WSAStartup(MAKEWORD(2,2), &wsa);

    s = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
    if (s == INVALID_SOCKET) {
        printf("socket error=%d\n", WSAGetLastError());
        return 1;
    }

    if (enableBcast) {
        int opt = 1;
        if (setsockopt(s, SOL_SOCKET, SO_BROADCAST,
                       (char*)&opt, sizeof(opt)) == SOCKET_ERROR) {
            printf("setsockopt(SO_BROADCAST) error=%d\n", WSAGetLastError());
        } else {
            printf("[SEND] SO_BROADCAST ON\n");
        }
    } else {
        printf("[SEND] SO_BROADCAST OFF\n");
    }

    addr.sin_family      = AF_INET;
    addr.sin_port        = htons(port);
    addr.sin_addr.s_addr = inet_addr("255.255.255.255");

    int n = sendto(s, msg, (int)strlen(msg), 0,
                   (SOCKADDR*)&addr, sizeof(addr));
    if (n == SOCKET_ERROR) {
        printf("sendto error=%d\n", WSAGetLastError());
    } else {
        printf("[SEND] sendto success: %d bytes\n", n);
    }

    closesocket(s);
    WSACleanup();
    return 0;
}

```
setsockopt옵션을 통해 SO_BROADCAST = 1 이라는 값을 보내면 목적지 주소를 255.255.255.255로 설정한다. 이 옵션 값을 1로 지정하지 않았을 때 윈도우에서 브로드캐스트 주소로보내면 오류가 난다.



## 3. keepalive_client_win

```cpp
// keepalive_client_win.cpp
#include "keepalive_win.h"
#include <ws2tcpip.h>

int main(int argc, char *argv[]) {
    WSADATA wsa;
    SOCKET s;
    struct sockaddr_in addr;
    const char *ip = "127.0.0.1";
    int port = 50003;
    char buf[512];

    if (argc >= 2) ip   = argv[1];
    if (argc >= 3) port = atoi(argv[2]);

    WSAStartup(MAKEWORD(2,2), &wsa);

    s = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (s == INVALID_SOCKET) {
        printf("socket error=%d\n", WSAGetLastError());
        return 1;
    }

    // 클라이언트 쪽에도 keepalive 설정
    set_keepalive(s, 10000, 3000);

    addr.sin_family      = AF_INET;
    addr.sin_port        = htons(port);
    addr.sin_addr.s_addr = inet_addr(ip);

    if (connect(s, (SOCKADDR*)&addr, sizeof(addr)) == SOCKET_ERROR) {
        printf("connect error=%d\n", WSAGetLastError());
        closesocket(s);
        WSACleanup();
        return 1;
    }

    printf("[CLIENT] Connected. Type lines to send. (Ctrl+Z+Enter to end)\n");

    while (fgets(buf, sizeof(buf), stdin)) {
        int n = send(s, buf, (int)strlen(buf), 0);
        if (n == SOCKET_ERROR) {
            printf("[CLIENT] send error=%d\n", WSAGetLastError());
            break;
        }
    }

    closesocket(s);
    WSACleanup();
    return 0;
}

```
setsockopt옵션을 통해 SO_KEEPALIVE = 1 이라는 값을 보내면 해당 연결해 대한 Keep_alive기능을 켠다. Keep_alive기능이란 일정시간동안 통신이 없을때, 프로브라는 패킷을 보내 상대가 유효한 상태인지를 확인하는 기능이다.

SIO_KEEPALIVE_VALS는 아무 통신이 없을 때 처음 KeepAlive를 보낼때 까지의 대기 시간이다.



## 4. linger_client + 5.linger_server

```cpp
// linger_client.cpp
#define _WINSOCK_DEPRECATED_NO_WARNINGS
#include <winsock2.h>
#include <ws2tcpip.h>
#include <stdio.h>

//#pragma comment(lib, "ws2_32.lib")

int main(int argc, char *argv[]) {
    WSADATA wsa;
    SOCKET s;
    struct sockaddr_in addr;
    char buf[4096];
    const char *ip = "127.0.0.1";
    int port = 50002;

    if (argc >= 2) ip   = argv[1];
    if (argc >= 3) port = atoi(argv[2]);

    WSAStartup(MAKEWORD(2,2), &wsa);

    s = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (s == INVALID_SOCKET) {
        printf("socket error=%d\n", WSAGetLastError());
        return 1;
    }

    addr.sin_family      = AF_INET;
    addr.sin_port        = htons(port);
    addr.sin_addr.s_addr = inet_addr(ip);

    if (connect(s, (SOCKADDR*)&addr, sizeof(addr)) == SOCKET_ERROR) {
        printf("connect error=%d\n", WSAGetLastError());
        closesocket(s);
        WSACleanup();
        return 1;
    }

    printf("[CLIENT] Connected. Receiving data...\n");
    while (1) {
        int n = recv(s, buf, sizeof(buf), 0);
        if (n > 0) {
            // 일부만 읽고 버려도 됨
            // printf(".");
        } else if (n == 0) {
            printf("\n[CLIENT] recv=0 (graceful close)\n");
            break;
        } else {
            printf("\n[CLIENT] recv error=%d\n", WSAGetLastError());
            break;
        }
    }

    closesocket(s);
    WSACleanup();
    return 0;
}


// linger_server.cpp
#define _WINSOCK_DEPRECATED_NO_WARNINGS
#include <winsock2.h>
#include <ws2tcpip.h>
#include <stdio.h>
#include <time.h>

//#pragma comment(lib, "ws2_32.lib")

int main(int argc, char *argv[]) {
    WSADATA wsa;
    SOCKET listenSock, clientSock;
    struct sockaddr_in addr, cliaddr;
    int port = 50002;
    int mode = 0; // 0: LINGER OFF, 1: ON(0sec, RST), 2: ON(5sec)
    char buf[4096];
    int i;

    if (argc >= 2) mode = atoi(argv[1]);
    if (argc >= 3) port = atoi(argv[2]);

    WSAStartup(MAKEWORD(2,2), &wsa);

    listenSock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (listenSock == INVALID_SOCKET) {
        printf("socket error=%d\n", WSAGetLastError());
        return 1;
    }

    int opt = 1;
    setsockopt(listenSock, SOL_SOCKET, SO_REUSEADDR, (char*)&opt, sizeof(opt));

    addr.sin_family      = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    addr.sin_port        = htons(port);

    bind(listenSock, (SOCKADDR*)&addr, sizeof(addr));
    listen(listenSock, 1);
    printf("[SERVER] Listening on port %d (mode=%d)\n", port, mode);

    int clen = sizeof(cliaddr);
    clientSock = accept(listenSock, (SOCKADDR*)&cliaddr, &clen);
    if (clientSock == INVALID_SOCKET) {
        printf("accept error=%d\n", WSAGetLastError());
        return 1;
    }
    printf("[SERVER] Client connected.\n");

    // 클라이언트가 천천히 받도록 하기 위해 잠깐 sleep 해도 좋음
    memset(buf, 'A', sizeof(buf));

    printf("[SERVER] Sending large data...\n");
    for (i = 0; i < 2000; i++) { // 2000 * 4096 bytes
        int n = send(clientSock, buf, sizeof(buf), 0);
        if (n == SOCKET_ERROR) {
            printf("send error=%d\n", WSAGetLastError());
            break;
        }
    }
    printf("[SERVER] Done send, now set SO_LINGER and close.\n");

    struct linger ling;
    if (mode == 0) {
        ling.l_onoff  = 0;
        ling.l_linger = 0;
        printf("[SERVER] SO_LINGER OFF\n");
    } else if (mode == 1) {
        ling.l_onoff  = 1;
        ling.l_linger = 0;
        printf("[SERVER] SO_LINGER ON, 0 sec (RST)\n");
    } else {
        ling.l_onoff  = 1;
        ling.l_linger = 5;
        printf("[SERVER] SO_LINGER ON, 5 sec\n");
    }
    setsockopt(clientSock, SOL_SOCKET, SO_LINGER,
               (char*)&ling, sizeof(ling));

    clock_t t1 = clock();
    int r = closesocket(clientSock);
    clock_t t2 = clock();

    double elapsed = (double)(t2 - t1) / CLOCKS_PER_SEC;
    printf("[SERVER] closesocket() ret=%d, elapsed=%.3f sec\n", r, elapsed);

    closesocket(listenSock);
    WSACleanup();
    return 0;
}

```
setsockopt옵션을 통해 SO_REUSEADDR라는 값을 보내면 해당 서버를 재실행할 때 같은 포트를 바로 다시 쓸 수 있게 해주는 옵션이다. (가끔 서버 소켓이 완전히 정리되지 않고 남아있는 오류가 발생할 수도 있기 때문)

setsockopt옵션을 통해 SO_LINGER라는 값은 여러 옵션으로 나눈다. 뒤에 들어가는 파라미터 값을 통해 정할 수 있다.
RST: 소켓을 닫을 때 바로 리턴하여 강제로 0초만에 끊는다.
FIN: 소켓을 닫을 때 입력된 초를 기다렸다가 끊는다.



## 6. reuseaddr_client + 7.reuseaddr_server

```cpp
// reuseaddr_client.cpp
#define _WINSOCK_DEPRECATED_NO_WARNINGS
#include <winsock2.h>
#include <ws2tcpip.h>
#include <stdio.h>

//#pragma comment(lib, "ws2_32.lib")

int main(int argc, char *argv[]) {
    WSADATA wsa;
    SOCKET s;
    struct sockaddr_in addr;
    const char *serverIp = "127.0.0.1";
    int port = 50000;

    if (argc >= 2) serverIp = argv[1];
    if (argc >= 3) port     = atoi(argv[2]);

    WSAStartup(MAKEWORD(2,2), &wsa);

    s = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (s == INVALID_SOCKET) {
        printf("socket error=%d\n", WSAGetLastError());
        return 1;
    }

    addr.sin_family = AF_INET;
    addr.sin_port   = htons(port);
    addr.sin_addr.s_addr = inet_addr(serverIp);

    if (connect(s, (SOCKADDR*)&addr, sizeof(addr)) == SOCKET_ERROR) {
        printf("connect error=%d\n", WSAGetLastError());
        closesocket(s);
        WSACleanup();
        return 1;
    }

    printf("[CLIENT] Connected. Closing immediately.\n");
    closesocket(s);
    WSACleanup();
    return 0;
}


// reuseaddr_server.cpp
#define _WINSOCK_DEPRECATED_NO_WARNINGS
#include <winsock2.h>
#include <ws2tcpip.h>
#include <stdio.h>

//#pragma comment(lib, "ws2_32.lib")

int main(int argc, char *argv[]) {
    WSADATA wsa;
    SOCKET listenSock = INVALID_SOCKET, clientSock = INVALID_SOCKET;
    struct sockaddr_in addr, cliaddr;
    int optval;
    int reuse = 0;  // 기본: OFF
    int port = 50000;

    if (argc >= 2) reuse = atoi(argv[1]); // 0 or 1
    if (argc >= 3) port  = atoi(argv[2]);

    WSAStartup(MAKEWORD(2,2), &wsa);

    listenSock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (listenSock == INVALID_SOCKET) {
        printf("socket error=%d\n", WSAGetLastError());
        return 1;
    }

    if (reuse) {
        optval = 1;
        if (setsockopt(listenSock, SOL_SOCKET, SO_REUSEADDR,
                       (char*)&optval, sizeof(optval)) == SOCKET_ERROR) {
            printf("setsockopt(SO_REUSEADDR) error=%d\n", WSAGetLastError());
        } else {
            printf("[SERVER] SO_REUSEADDR ON\n");
        }
    } else {
        printf("[SERVER] SO_REUSEADDR OFF\n");
    }

    addr.sin_family      = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    addr.sin_port        = htons(port);

    if (bind(listenSock, (SOCKADDR*)&addr, sizeof(addr)) == SOCKET_ERROR) {
        printf("bind error=%d\n", WSAGetLastError());
        closesocket(listenSock);
        WSACleanup();
        return 1;
    }

    if (listen(listenSock, 1) == SOCKET_ERROR) {
        printf("listen error=%d\n", WSAGetLastError());
        closesocket(listenSock);
        WSACleanup();
        return 1;
    }

    printf("[SERVER] Listening on port %d ... (press Enter after one client connects)\n", port);

    int clen = sizeof(cliaddr);
    clientSock = accept(listenSock, (SOCKADDR*)&cliaddr, &clen);
    if (clientSock == INVALID_SOCKET) {
        printf("accept error=%d\n", WSAGetLastError());
        closesocket(listenSock);
        WSACleanup();
        return 1;
    }
    printf("[SERVER] Client connected. Press Enter to close server.\n");
    getchar();

    closesocket(clientSock);
    closesocket(listenSock);
    WSACleanup();
    printf("[SERVER] Exit.\n");
    return 0;
}

```
setsockopt옵션을 통해 SO_REUSEADDR라는 값을 보내면 해당 서버를 재실행할 때 같은 포트를 바로 다시 쓸 수 있게 해주는 옵션이다. (가끔 서버 소켓이 완전히 정리되지 않고 남아있는 오류가 발생할 수도 있기 때문)

## 8. sol_socket_winsock.cpp

```cpp
/* 
 * sol_socket_winsock.cpp
 *
 * Windows / Winsock2 에서 SOL_SOCKET 레벨 socket option 을
 * getsockopt / setsockopt 으로 읽고/설정하는 검증용 코드
 *
 *   sol_socket_winsock.exe
 */

#define _WINSOCK_DEPRECATED_NO_WARNINGS

#include <winsock2.h>
#include <ws2tcpip.h>
#include <stdio.h>

//#pragma comment(lib, "ws2_32.lib")

void die(const char *msg) {
    DWORD err = WSAGetLastError();
    fprintf(stderr, "%s failed with error %lu\n", msg, err);
}

/* 정수/불리언 옵션 출력 */
void print_int_opt(SOCKET s, int optname, const char *name) {
    int val = 0;
    int len = sizeof(val);

    if (getsockopt(s, SOL_SOCKET, optname, (char*)&val, &len) == SOCKET_ERROR) {
        printf("%s: getsockopt 실패 (err=%d)\n", name, WSAGetLastError());
        return;
    }
    printf("  %-15s = %d\n", name, val);
}

/* SO_LINGER 전용 출력 */
void print_linger_opt(SOCKET s) {
    struct linger ling;
    int len = sizeof(ling);

    if (getsockopt(s, SOL_SOCKET, SO_LINGER, (char*)&ling, &len) == SOCKET_ERROR) {
        printf("SO_LINGER: getsockopt 실패 (err=%d)\n", WSAGetLastError());
        return;
    }
    printf("  %-15s = onoff=%d, linger=%d\n", "SO_LINGER", ling.l_onoff, ling.l_linger);
}

int main(void) {
    WSADATA wsaData;
    SOCKET sock = INVALID_SOCKET;
    int optval;
    struct linger ling;

    /* 1. Winsock 초기화 */
    if (WSAStartup(MAKEWORD(2,2), &wsaData) != 0) {
        fprintf(stderr, "WSAStartup 실패\n");
        return 1;
    }

    /* 2. TCP 스트림 소켓 생성 */
    sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (sock == INVALID_SOCKET) {
        die("socket");
        WSACleanup();
        return 1;
    }

    printf("=== SOL_SOCKET 옵션 기본값 ===\n");
    print_int_opt(sock, SO_REUSEADDR, "SO_REUSEADDR");
    print_int_opt(sock, SO_BROADCAST, "SO_BROADCAST");
    print_int_opt(sock, SO_KEEPALIVE, "SO_KEEPALIVE");
    print_int_opt(sock, SO_RCVBUF,    "SO_RCVBUF");
    print_int_opt(sock, SO_SNDBUF,    "SO_SNDBUF");
    print_linger_opt(sock);
    print_int_opt(sock, SO_ERROR,     "SO_ERROR");
    print_int_opt(sock, SO_TYPE,      "SO_TYPE");

    /* SO_REUSEADDR 설정 후 다시 조회 */
    printf("\n=== SO_REUSEADDR 켜기 ===\n");
    optval = 1;
    if (setsockopt(sock, SOL_SOCKET, SO_REUSEADDR, (char*)&optval, sizeof(optval)) == SOCKET_ERROR) {
        printf("SO_REUSEADDR setsockopt 실패 (err=%d)\n", WSAGetLastError());
    }
    print_int_opt(sock, SO_REUSEADDR, "SO_REUSEADDR");

    /* SO_BROADCAST, SO_KEEPALIVE 설정 후 다시 조회 */
    printf("\n=== SO_BROADCAST, SO_KEEPALIVE 켜기 ===\n");
    optval = 1;
    if (setsockopt(sock, SOL_SOCKET, SO_BROADCAST, (char*)&optval, sizeof(optval)) == SOCKET_ERROR) {
        printf("SO_BROADCAST setsockopt 실패 (err=%d)\n", WSAGetLastError());
    }
    if (setsockopt(sock, SOL_SOCKET, SO_KEEPALIVE, (char*)&optval, sizeof(optval)) == SOCKET_ERROR) {
        printf("SO_KEEPALIVE setsockopt 실패 (err=%d)\n", WSAGetLastError());
    }
    print_int_opt(sock, SO_BROADCAST, "SO_BROADCAST");
    print_int_opt(sock, SO_KEEPALIVE, "SO_KEEPALIVE");

    /* SO_RCVBUF / SO_SNDBUF 크게 설정 후 확인 */
    printf("\n=== SO_RCVBUF / SO_SNDBUF 값을 크게 설정 ===\n");
    optval = 1 << 20;  /* 1MB 요청 (커널이 조정할 수 있음) */
    if (setsockopt(sock, SOL_SOCKET, SO_RCVBUF, (char*)&optval, sizeof(optval)) == SOCKET_ERROR) {
        printf("SO_RCVBUF setsockopt 실패 (err=%d)\n", WSAGetLastError());
    }
    if (setsockopt(sock, SOL_SOCKET, SO_SNDBUF, (char*)&optval, sizeof(optval)) == SOCKET_ERROR) {
        printf("SO_SNDBUF setsockopt 실패 (err=%d)\n", WSAGetLastError());
    }
    print_int_opt(sock, SO_RCVBUF, "SO_RCVBUF");
    print_int_opt(sock, SO_SNDBUF, "SO_SNDBUF");

    /* SO_LINGER 설정 후 확인 */
    printf("\n=== SO_LINGER 설정 (on, 5초) ===\n");
    ling.l_onoff = 1;
    ling.l_linger = 5;
    if (setsockopt(sock, SOL_SOCKET, SO_LINGER, (char*)&ling, sizeof(ling)) == SOCKET_ERROR) {
        printf("SO_LINGER setsockopt 실패 (err=%d)\n", WSAGetLastError());
    }
    print_linger_opt(sock);

    /* 읽기 전용 성격 옵션 재확인 */
    printf("\n=== 읽기 전용 옵션 (SO_ERROR, SO_TYPE) 재확인 ===\n");
    print_int_opt(sock, SO_ERROR, "SO_ERROR");
    print_int_opt(sock, SO_TYPE,  "SO_TYPE");

    closesocket(sock);
    WSACleanup();
    return 0;
}

```
setsockopt옵션을 통해 SO_REUSEADDR 값을 보내면 해당 서버를 재실행할 때 같은 포트를 바로 다시 쓸 수 있게 해주는 옵션이다. (가끔 서버 소켓이 완전히 정리되지 않고 남아있는 오류가 발생할 수도 있기 때문)

setsockopt옵션을 통해 SO_BROADCAST = 1 이라는 값을 보내면 목적지 주소를 255.255.255.255로 설정한다. 이 옵션 값을 1로 지정하지 않았을 때 윈도우에서 브로드캐스트 주소로보내면 오류가 난다.

setsockopt옵션을 통해 SO_RCVBUF라는 값은 수신쪽 버퍼 크기를 나타낸다.

setsockopt옵션을 통해 SSO_SNDBUF라는 값은 송신쪽 버퍼 크기를 나타낸다.

setsockopt옵션을 통해 SO_LINGER라는 값은 여러 옵션으로 나눈다. 뒤에 들어가는 파라미터 값을 통해 정할 수 있다.
RST: 소켓을 닫을 때 바로 리턴하여 강제로 0초만에 끊는다.
FIN: 소켓을 닫을 때 입력된 초를 기다렸다가 끊는다.

setsockopt옵션을 통해 SO_ERROR라는 값은 해당 소켓에 대한 최근에 발생한 오류 코드를 가져오는 옵션이다. 읽기전용 옵션이다.

setsockopt옵션을 통해 SO_TYPE라는 값은 이 소켓이 TCP인지, UDP인지 소켓타입을 알려주는 옵션이다. (SOCK_STREAM, SOCK_DGRAM) 읽기전용 옵션이다.
