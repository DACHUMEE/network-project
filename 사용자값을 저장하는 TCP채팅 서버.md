# 프로그램 코드와 설명

## 해당 코드는 TCP서버에서 연결 정보를 관리하는 자료 저장소 구현하였다. Vector를 static으로 선언하여 공유되는 메모리공간을 할당하였고, UpdateClient 함수를 통해 사용자가 접속하고 접속을 종료할때 상태를 저장하고 관리할 수 있게 구현하였다. 접속이 종료되면 사용자의 정보는 삭제된다.

```cpp
#include "..\..\Common.h"
#include <string>
#include <vector>
#include <algorithm>

using namespace std;

#define SERVERPORT 9000
#define BUFSIZE    512

struct ClientInfor {
	SOCKET sock;              
	string name;
	IN_ADDR ip; 
	int port;                 
};

static vector<ClientInfor> vect;

void UpdateClient(SOCKET sock, sockaddr_in addrin, string sType) {

	ClientInfor ci;
	ci.sock = sock;
	ci.ip = addrin.sin_addr;
	ci.port = ntohs(addrin.sin_port);

	char addr[INET_ADDRSTRLEN];
	inet_ntop(AF_INET, &addrin.sin_addr, addr, sizeof(addr));

	string sTypeNm = "";

	if (sType == "INSERT") {
		vect.push_back(ci);

		sTypeNm = "연결성공";
	}
	else if (sType == "DELETE") {
		vector<ClientInfor>::iterator it;
		it = remove_if(vect.begin(), vect.end(), [sock](ClientInfor& c) { return c.sock == sock; });

		vect.erase(it, vect.end());

		sTypeNm = "연결종료";
	}
	else {
		printf("알수 없는 Type값 오류\n");
	}

	printf_s("사용자 데이터 변경-> IP주소: %s, PORT번호: %d, 접속상태: %s\n", addr, ci.port, sTypeNm.c_str());
}

// 클라이언트와 데이터 통신
DWORD WINAPI ProcessClient(LPVOID arg)
{
	int retval;
	SOCKET client_sock = (SOCKET)arg;
	struct sockaddr_in clientaddr;
	char addr[INET_ADDRSTRLEN];
	int addrlen;
	char buf[BUFSIZE + 1];

	// 클라이언트 정보 얻기
	addrlen = sizeof(clientaddr);
	getpeername(client_sock, (struct sockaddr *)&clientaddr, &addrlen);
	inet_ntop(AF_INET, &clientaddr.sin_addr, addr, sizeof(addr));

	while (1) {
		// 데이터 받기
		retval = recv(client_sock, buf, BUFSIZE, 0);
		if (retval == SOCKET_ERROR) {
			err_display("recv()");
			break;
		}
		else if (retval == 0)
			break;

		// 받은 데이터 출력
		buf[retval] = '\0';
		printf("[TCP/%s:%d] %s\n", addr, ntohs(clientaddr.sin_port), buf);

		// 데이터 보내기
		retval = send(client_sock, buf, retval, 0);
		if (retval == SOCKET_ERROR) {
			err_display("send()");
			break;
		}
	}

	UpdateClient(client_sock, clientaddr, "DELETE");
	// 소켓 닫기
	closesocket(client_sock);

	printf("[TCP 서버] 클라이언트 종료: IP 주소=%s, 포트 번호=%d\n",
		addr, ntohs(clientaddr.sin_port));
	return 0;
}

int main(int argc, char *argv[])
{

	int retval;

	// 윈속 초기화
	WSADATA wsa;
	if (WSAStartup(MAKEWORD(2, 2), &wsa) != 0)
		return 1;

	// 소켓 생성
	SOCKET listen_sock = socket(AF_INET, SOCK_STREAM, 0);
	if (listen_sock == INVALID_SOCKET) err_quit("socket()");

	// bind()
	struct sockaddr_in serveraddr;
	memset(&serveraddr, 0, sizeof(serveraddr));
	serveraddr.sin_family = AF_INET;
	serveraddr.sin_addr.s_addr = htonl(INADDR_ANY);
	serveraddr.sin_port = htons(SERVERPORT);
	retval = bind(listen_sock, (struct sockaddr *)&serveraddr, sizeof(serveraddr));
	if (retval == SOCKET_ERROR) err_quit("bind()");

	// listen()
	retval = listen(listen_sock, SOMAXCONN);
	if (retval == SOCKET_ERROR) err_quit("listen()");

	// 데이터 통신에 사용할 변수
	SOCKET client_sock;
	struct sockaddr_in clientaddr;
	int addrlen;
	HANDLE hThread;

	while (1) {
		// accept()
		addrlen = sizeof(clientaddr);
		client_sock = accept(listen_sock, (struct sockaddr *)&clientaddr, &addrlen);
		if (client_sock == INVALID_SOCKET) {
			err_display("accept()");
			break;
		}

		// 접속한 클라이언트 정보 출력
		char addr[INET_ADDRSTRLEN];
		inet_ntop(AF_INET, &clientaddr.sin_addr, addr, sizeof(addr));
		printf("\n[TCP 서버] 클라이언트 접속: IP 주소=%s, 포트 번호=%d\n",
			addr, ntohs(clientaddr.sin_port));

		UpdateClient(client_sock, clientaddr, "INSERT");

		// 스레드 생성
		hThread = CreateThread(NULL, 0, ProcessClient,
			(LPVOID)client_sock, 0, NULL);
		if (hThread == NULL) { closesocket(client_sock); }
		else { CloseHandle(hThread); }
	}

	// 소켓 닫기
	closesocket(listen_sock);

	// 윈속 종료
	WSACleanup();
	return 0;
}

```
결과:
[TCP 서버] 클라이언트 접속: IP 주소=000.0.0.1, 포트 번호=56124
사용자 데이터 변경-> IP주소: 000.0.0.1, PORT번호: 56124, 접속상태: 연결성공
[TCP/127.0.0.1:56124] gd
[TCP/127.0.0.1:56124] ㅎㅇ
[recv()] 현재 연결은 원격 호스트에 의해 강제로 끊겼습니다.

사용자 데이터 변경-> IP주소: 000.0.0.1, PORT번호: 56124, 접속상태: 연결종료
[TCP 서버] 클라이언트 종료: IP 주소=000.0.0.1, 포트 번호=56124