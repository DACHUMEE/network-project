# 프로그램 코드와 설명

## 해당 코드는 TCP서버에서 p2p형식으로 메시지를 주고받는 서버 코드이다. 기존 UDP소켓을 TCP소켓으로 변경하고, bind나 listen과정 없이 coonet만 추가하여 구현한 간단하 tcp 서버 코드이다. 

```cpp
#include <stdio.h>
#include <string.h>
#include <winsock2.h>
#include <ws2tcpip.h>
#include <conio.h>

#pragma comment(lib, "ws2_32.lib")

#define PORT1  9001   // 내가 서버에 보낼때 쓰는 포트
#define PORT2  9002   // 다른 사람이 나한테 보낼때 쓰는 포트
#define BUFSIZE    512

char* SERVERIP = (char*)"192.168.219.100";

int main() {

    WSADATA wsa;
    WSAStartup(MAKEWORD(2, 2), &wsa);

    // 클라이언트 역할 소켓
    SOCKET sockSend = socket(AF_INET, SOCK_STREAM, 0);
    sockaddr_in addr1;
    addr1.sin_family = AF_INET;
    addr1.sin_addr.s_addr = htonl(INADDR_ANY);
    addr1.sin_port = htons(PORT1);
    bind(sockSend, (sockaddr*)&addr1, sizeof(addr1));

    // 서버 역할 소켓
    SOCKET sockRecv = socket(AF_INET, SOCK_STREAM, 0);
    sockaddr_in addr2;
    addr2.sin_family = AF_INET;
    addr2.sin_addr.s_addr = htonl(INADDR_ANY);
    addr2.sin_port = htons(PORT2);
    bind(sockRecv, (sockaddr*)&addr2, sizeof(addr2));
    listen(sockRecv, 5);

    // 보낼 서버 주소 (임의로 설정)
    sockaddr_in srvAddr;
    srvAddr.sin_family = AF_INET;
    inet_pton(AF_INET, SERVERIP, &srvAddr.sin_addr);
    srvAddr.sin_port = htons(9000); // 상대 서버 포트
    connect(sockSend, (sockaddr*)&srvAddr, sizeof(srvAddr));

    // 소켓을 관리하는 클래스 선언
    fd_set fdList;

    printf("p2p 테스트\n");

    SOCKET peerSock = INVALID_SOCKET;

    while (1)
    {
        // FD_ZERO: fdList 비우기 (초기화)
        FD_ZERO(&fdList);

        // FD_SET: fdList에 소켓목록 추가 
        FD_SET(sockSend, &fdList);
        FD_SET(sockRecv, &fdList);
        if (peerSock != INVALID_SOCKET) FD_SET(peerSock, &fdList);

        timeval tv;
        tv.tv_sec = 0;
        tv.tv_usec = 100 * 1000;

        int ret = select(0, &fdList, NULL, NULL, &tv);

        // 키보드 입력이 존재하면 서버로 보내기
        if (_kbhit())
        {
            char msg[BUFSIZE];
            fgets(msg, BUFSIZE, stdin);

            int len = (int)strlen(msg);
            if (len > 0 && msg[len - 1] == '\n') msg[len - 1] = 0;

            send(sockSend, msg, (int)strlen(msg), 0);

            printf("서버로 보냄: %s\n", msg);
        }

        // 서버에서 보내준 응답(sockSend에 도착)
        if (FD_ISSET(sockSend, &fdList))
        {
            char buf[BUFSIZE + 1];

            int n = recv(sockSend, buf, BUFSIZE, 0);
            if (n <= 0) break;

            buf[n] = 0;
            printf("서버 응답: %s\n", buf);
        }

        // 다른 클라이언트에서 이쪽으로 요청을 보내는 것을 확인
        if (FD_ISSET(sockRecv, &fdList))
        {
            sockaddr_in from2;
            int fsize2 = sizeof(from2);

            SOCKET s = accept(sockRecv, (sockaddr*)&from2, &fsize2);
            if (s != INVALID_SOCKET)
            {
                if (peerSock != INVALID_SOCKET) closesocket(peerSock);
                peerSock = s;

                char ip[30];
                inet_ntop(AF_INET, &from2.sin_addr, ip, sizeof(ip));

                printf("[클라이언트 %s:%d] 연결됨\n",ip, ntohs(from2.sin_port));
            }
        }

        if (peerSock != INVALID_SOCKET && FD_ISSET(peerSock, &fdList))
        {
            char buf[BUFSIZE + 1];

            int n = recv(peerSock, buf, BUFSIZE, 0);
            if (n <= 0)
            {
                closesocket(peerSock);
                peerSock = INVALID_SOCKET;
                continue;
            }

            buf[n] = 0;
        }
    }

    if (peerSock != INVALID_SOCKET) closesocket(peerSock);
    closesocket(sockSend);
    closesocket(sockRecv);
    WSACleanup();
}
```

