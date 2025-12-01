# 개념과 프로그램 코드

## p2p란?
-> P2P(Peer-to-Peer)란, 프로그램이 클라이언트인 동시에 서버 기능을 하는 구조이며 각 프로그램을 peer단위라고 칭한다.


## 해당 코드는 클라이언트와 서버 기능을 동시에 하는 UDP p2p이다. 소켓을 관리하는 fd_set클래스 변수를 선언하여 각 기능별로 소켓을 저장한다. Select를 이용하여 각 소켓이 구분되게 동작되도록 구현한다.

```cpp
#include <stdio.h>
#include <winsock2.h>
#include <ws2tcpip.h>

#pragma comment(lib, "ws2_32.lib")

#define PORT1  9001   // 내가 서버에 보낼때 쓰는 포트
#define PORT2  9002   // 다른 사람이 나한테 보낼때 쓰는 포트
#define BUFSIZE    512

char* SERVERIP = (char*)"192.168.219.100";

int main() {

    WSADATA wsa;
    WSAStartup(MAKEWORD(2, 2), &wsa);

    // 클라이언트 역할 소켓
    SOCKET sockSend = socket(AF_INET, SOCK_DGRAM, 0);
    sockaddr_in addr1;
    addr1.sin_family = AF_INET;
    addr1.sin_addr.s_addr = htonl(INADDR_ANY);
    addr1.sin_port = htons(PORT1);
    bind(sockSend, (sockaddr*)&addr1, sizeof(addr1));

    // 서버 역할 소켓
    SOCKET sockRecv = socket(AF_INET, SOCK_DGRAM, 0);
    sockaddr_in addr2 ;
    addr2.sin_family = AF_INET;
    addr2.sin_addr.s_addr = htonl(INADDR_ANY);
    addr2.sin_port = htons(PORT2);
    bind(sockRecv, (sockaddr*)&addr2, sizeof(addr2));

    // 보낼 서버 주소 (임의로 설정)
    sockaddr_in srvAddr;
    srvAddr.sin_family = AF_INET;
    inet_pton(AF_INET, SERVERIP, &srvAddr.sin_addr);
    srvAddr.sin_port = htons(9000); // 상대 서버 포트

    // 소켓을 관리하는 클래스 선언
    fd_set fdList;

    printf("p2p 테스트\n");

    while (1)
    {
        // FD_ZERO: fdList 비우기 (초기화)
        FD_ZERO(&fdList);

        // FD_SET: fdList에 소켓목록 추가 
        FD_SET(sockSend, &fdList);
        FD_SET(sockRecv, &fdList);
        FD_SET(0, &fdList); // 0은 표준입력을 나타내어 키보드 입력을 나타냄.

        int ret = select(0, &fdList, NULL, NULL, NULL);

        // 키보드 입력이 존재하면 서버로 보내기
        if (FD_ISSET(0, &fdList))
        {
            char msg[BUFSIZE];
            fgets(msg, BUFSIZE, stdin);

            int len = (int)strlen(msg);
            if (msg[len - 1] == '\n') msg[len - 1] = 0;

            sendto(sockSend, msg, strlen(msg), 0,
                (sockaddr*)&srvAddr, sizeof(srvAddr));

            printf("서버로 보냄: %s\n", msg);
        }

        // 서버에서 보내준 응답(sockSend에 도착)
        if (FD_ISSET(sockSend, &fdList))
        {
            char buf[BUFSIZE];
            sockaddr_in from;
            int fsize = sizeof(from);

            int n = recvfrom(sockSend, buf, BUFSIZE, 0,
                (sockaddr*)&from, &fsize);

            buf[n] = 0;
            printf("[서버 응답] %s\n", buf);
        }

        // 다른 클라이언트에서 이쪽으로 요청을 보내는 것을 확인
        if (FD_ISSET(sockRecv, &fdList))
        {
            char buf[BUFSIZE];
            sockaddr_in from2;
            int fsize2 = sizeof(from2);

            int n = recvfrom(sockRecv, buf, BUFSIZE, 0,
                (sockaddr*)&from2, &fsize2);

            buf[n] = 0;

            char ip[30];
            inet_ntop(AF_INET, &from2.sin_addr, ip, sizeof(ip));

            printf("[다른 클라이언트 %s:%d] %s\n",
                ip, ntohs(from2.sin_port), buf);
        }
    }

    closesocket(sockSend);
    closesocket(sockRecv);
    WSACleanup();
}

```