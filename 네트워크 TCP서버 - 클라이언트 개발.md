# 프로그램 코드와 설명

## 현재 구현하려고 하는 프로그램은 클라이언트로 부터 list라는 문자열을 받으면 서버는 프로젝트 경로에 있는 txt 파일과 png 파일 목록을 전송한다.클라이언트는 수신한 목록 중 하 받고싶은 파일 이름을 전달하면 서버는 파일을 읽어 클라이언트에 전송한다. 클라이언트 파일을 수신한 후 저장한다.

## 1. 서버

```cpp
// server.cpp : 간단한 TCP 파일 전송 서버
// 빌드: cl /EHsc server.cpp ws2_32.lib

#define _WINSOCK_DEPRECATED_NO_WARNINGS
#define BUFSIZE    512

#include <winsock2.h>
#include <iostream>
#include <fstream>
#include <windows.h> // 있는 파일을 가져오기 위해 추가

using namespace std;

#pragma comment(lib, "ws2_32.lib")

string GetFilename() {

    string sFile = "";
    WIN32_FIND_DATAA wData; // 파일정보를 담는 구조체 변수
    HANDLE hFind = FindFirstFileA(".\\*", &wData); // 현재 디렉터리에 모든 파일 검색 (.\\은 현재 디렉토리, *은 모든 파일, 파일정보를 담을 곳)

    // 정상적으로 파일이 입력됐을 때
    if (hFind != INVALID_HANDLE_VALUE) {
        while (FindNextFileA(hFind, &wData) != 0)
        {
            string sNmae = wData.cFileName;

            if (sNmae.size() >= 3) {
                std::string sLast3;
                sLast3 += (char)tolower(sNmae[sNmae.size() - 3]);
                sLast3 += (char)tolower(sNmae[sNmae.size() - 2]);
                sLast3 += (char)tolower(sNmae[sNmae.size() - 1]);

                if (sLast3 == "png" || sLast3 == "txt") {
                    sFile += sNmae;
                    sFile += "\n";
                }
            }
        }
        FindClose(hFind);
    }
    else {
        sFile = "";
    }
    return sFile;
}

int main() {
    WSADATA wsaData;
    WSAStartup(MAKEWORD(2, 2), &wsaData); // WinSock 초기화

    SOCKET serverSock = socket(AF_INET, SOCK_STREAM, 0); // TCP 소켓 생성

    sockaddr_in serverAddr;
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_addr.s_addr = INADDR_ANY;  // 모든 IP에서 접속 허용
    serverAddr.sin_port = htons(9000);        // 포트 번호 9000 사용

    // 소켓에 주소 바인딩
    bind(serverSock, (SOCKADDR*)&serverAddr, sizeof(serverAddr));

    // 클라이언트 접속 대기
    listen(serverSock, 1);

    sockaddr_in clientAddr;
    int clientAddrSize = sizeof(clientAddr);
    SOCKET clientSock = accept(serverSock, (SOCKADDR*)&clientAddr, &clientAddrSize);

    int retval;
    int addrlen;
    char buf[BUFSIZE + 1];

    retval = recv(clientSock, buf, BUFSIZE, 0);

    // 현재 디렉터리의 png/txt 파일 목록 가져오기
    string sFileList = GetFilename();  // sFileList 안에 "파일이름1\n파일이름2\n..." 형태로 들어 있음

    if (sFileList != "") {
        int sent = send(clientSock, sFileList.c_str(), (int)sFileList.size(), 0);
    }
    else {
        // 조건에 맞는 파일이 하나도 없을 때
        const char* msg = "no file\n";
        send(clientSock, msg, (int)strlen(msg), 0);
    }

    retval = recv(clientSock, buf, BUFSIZE, 0);

    string sFileName(buf);

    printf_s("클라이언트 요청 파일: %s", sFileName);


    // 보낼 파일 열기
    ifstream file(sFileName, ios::binary);
    if (!file) {
        cerr << "[서버] 파일을 열 수 없습니다.\n";
        closesocket(clientSock);
        closesocket(serverSock);
        WSACleanup();
        return 1;
    }

    // 파일 내용을 읽어서 클라이언트로 전송
    char buffer[1024];
    while (!file.eof()) {
        file.read(buffer, sizeof(buffer));
        int bytesRead = file.gcount();
        send(clientSock, buffer, bytesRead, 0);
    }

    std::cout << "[서버] 파일 전송 완료!\n";

    // 정리
    file.close();

    closesocket(clientSock);
    closesocket(serverSock);
    WSACleanup();

    return 0;
}
```
현재 클라이언트에게 list라는 문자열을 받고, 디렉토리에 있는 파일 목록을 전해준다. GetFilename이라는 함수를 통해
프로젝트 경로에있는 파일 목록들을 읽고, 전달한다. 파일이름의 마지막 3글자를 체크하여 png, txt파일만 전송이 되도록 구현. 현재 클라이언트에게 파일 목록까지는 전달되지만, 파일 이름을 전달받을 때 오류가 난다.

## 2. 클라이언트 

```cpp
// client.cpp : 간단한 TCP 파일 수신 클라이언트
// 빌드: cl /EHsc client.cpp ws2_32.lib

#define _WINSOCK_DEPRECATED_NO_WARNINGS
#define BUFSIZE    512

#include <winsock2.h>
#include <iostream>
#include <fstream>
#pragma comment(lib, "ws2_32.lib")

using namespace std;

int main() {
    WSADATA wsaData;
    WSAStartup(MAKEWORD(2, 2), &wsaData); // WinSock 초기화

    SOCKET clientSock = socket(AF_INET, SOCK_STREAM, 0); // TCP 소켓 생성

    sockaddr_in serverAddr;
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_addr.s_addr = inet_addr("127.0.0.1"); // 서버 주소 (로컬)
    serverAddr.sin_port = htons(9000);                   // 서버 포트

    // 서버에 연결 시도
    if (connect(clientSock, (SOCKADDR*)&serverAddr, sizeof(serverAddr)) == SOCKET_ERROR) {
        std::cerr << "[클라이언트] 서버에 연결할 수 없습니다.\n";
        closesocket(clientSock);
        WSACleanup();
        return 1;
    }

    std::cout << "[클라이언트] 서버에 연결되었습니다.\n";

    printf_s("서버의 파일 목록을 받으려면 list라고 입력해 주세요. >");

    string sList;

    cin >> sList;

    if (sList != "list") {
        closesocket(clientSock);
        WSACleanup();
        return 1;
    }
    // 클라이언트가 list라고 입력함

    // 데이터 통신에 사용할 변수
    int retval;
    char buf[BUFSIZE + 1];
    int len;

    // 데이터 보내기
    retval = send(clientSock, sList.c_str(), (int)sList.size(), 0);

    printf_s("[TCP 클라이언트] %d바이트를 보냈습니다.\n", retval);

    retval = recv(clientSock, buf, retval, MSG_WAITALL);
    if (retval == 0) {
        printf_s("파일 목록을 받지 못했습니다.\n");
        closesocket(clientSock);
        WSACleanup();
        return 1;
    }

    buf[retval] = '\0';
    printf_s("서버파일목록 --------------------\n");
    printf_s("%s\n", buf);
    printf_s("---------------------------------\n\n");

    string sFile = "";

    printf_s("다운받을 파일 이름을 입력해 주세요.ex) filename.txt >");
    scanf_s("%s", sFile);

    // 데이터 보내기
    retval = send(clientSock, sFile.c_str(), (int)sFile.size(), 0);

    // 수신한 데이터를 파일로 저장
    std::ofstream outFile(sFile, std::ios::binary);
    char buffer[1024];
    int bytesReceived;

    while ((bytesReceived = recv(clientSock, buffer, sizeof(buffer), 0)) > 0) {
        outFile.write(buffer, bytesReceived);
    }

    std::cout << "[클라이언트] 파일 수신 완료!\n";

    // 정리
    outFile.close();
    closesocket(clientSock);
    WSACleanup();

    return 0;
}

```
현재 서버한테서 파일목록을 받고, 파일목록을 클라이언트 실행창 화면에 띄워지는 것 까지 구현이 완료되었다. 하지만 파일이름을 서버에게 전달시 예외가 발생했다고 뜨며 실행이 종료된다. 후에 더 확인하고 수정해야 하는 코드이다.

# 수정해야하는 부분

## 1. 예외가 발생했다고 하는 오류 처리
## 2. 동작이 오류없이 완료되는지 확인
## 3. 클라이언트 실행창에 확장자까지 포홤 되도록 수정
## 4. 클라이언트가 get을 통해 입력하고, 서버는 get을 뺀 파일명을 보내주도록 수정.