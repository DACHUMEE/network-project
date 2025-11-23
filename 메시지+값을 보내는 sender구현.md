# 프로그램 코드와 설명

## 해당 코드는 브로드캐스트를 통해 여러 메시지를 출력하는 수신자에게 특정 문자열 + 카운트 값을 보내도록 수정한 Sender코드다.

```cpp
#include "..\..\Common.h"

#define REMOTEIP   "255.255.255.255"
#define REMOTEPORT 9000
#define BUFSIZE    512

int main(int argc, char *argv[])
{
	int retval;

	// 윈속 초기화
	WSADATA wsa;
	if (WSAStartup(MAKEWORD(2, 2), &wsa) != 0)
		return 1;

	// 소켓 생성
	SOCKET sock = socket(AF_INET, SOCK_DGRAM, 0);
	if (sock == INVALID_SOCKET) err_quit("socket()");

	// 브로드캐스팅 활성화
	DWORD bEnable = 1;
	retval = setsockopt(sock, SOL_SOCKET, SO_BROADCAST,
		(const char *)&bEnable, sizeof(bEnable));
	if (retval == SOCKET_ERROR) err_quit("setsockopt()");

	// 소켓 주소 구조체 초기화
	struct sockaddr_in remoteaddr;
	memset(&remoteaddr, 0, sizeof(remoteaddr));
	remoteaddr.sin_family = AF_INET;
	inet_pton(AF_INET, REMOTEIP, &remoteaddr.sin_addr);
	remoteaddr.sin_port = htons(REMOTEPORT);

	// 데이터 통신에 사용할 변수
	char buf[BUFSIZE + 1];
	int len;

	// 숫자값을 기억하는 정수형 변수
	int iCnt = 0;

	// 브로드캐스트 데이터 보내기
	while (1) {
		// 데이터 입력
		printf("\n[보낼 데이터] ");
		/*if (fgets(buf, BUFSIZE + 1, stdin) == NULL)
			break;*/
		sprintf_s(buf, BUFSIZE + 1, "Hello World - %d", iCnt);

		// '\n' 문자 제거
		len = (int)strlen(buf);
		if (buf[len - 1] == '\n')
			buf[len - 1] = '\0';
		if (strlen(buf) == 0)
			break;

		// 데이터 보내기
		retval = sendto(sock, buf, (int)strlen(buf), 0,
			(struct sockaddr *)&remoteaddr, sizeof(remoteaddr));
		if (retval == SOCKET_ERROR) {
			err_display("sendto()");
			break;
		}
		printf("[UDP] %d바이트를 보냈습니다.\n", retval);

		iCnt++;
		Sleep(2000); // 2초마다 전송됨
	}

	// 소켓 닫기
	closesocket(sock);

	// 윈속 종료
	WSACleanup();
	return 0;
}

```
iCnt라는 카운트 값을 저장할 정수형 변수를 선언.

입력된 문자열을 보내는 fgets가 아닌 특정 문자열을 저장하는 sprintf_s함수를 사용하여 HelloWorld - n 으로 출력되도록 수정.

Sleep(2000);을 통해 2초 간격으로 메시지가 보내지며 카운트가 올라가도록 설정.

Sender 결과:

[보낼 데이터] [UDP] 15바이트를 보냈습니다.

[보낼 데이터] [UDP] 15바이트를 보냈습니다.

[보낼 데이터] [UDP] 15바이트를 보냈습니다.

[보낼 데이터] [UDP] 15바이트를 보냈습니다.

[보낼 데이터] [UDP] 15바이트를 보냈습니다.

[보낼 데이터] [UDP] 15바이트를 보냈습니다.

[보낼 데이터] [UDP] 15바이트를 보냈습니다.


Reciever 결과:
[UDP/아이피/포트] Hello World - 0
[UDP/아이피/포트] Hello World - 1
[UDP/아이피/포트] Hello World - 2
[UDP/아이피/포트] Hello World - 3
[UDP/아이피/포트] Hello World - 4
[UDP/아이피/포트] Hello World - 5

