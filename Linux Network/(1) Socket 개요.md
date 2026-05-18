## Socket 통신
Socket을 이용해, **서버-클라이언트** 간 데이터를 주고받는 양방향 연결 지향성 통신

![Socket 통신](https://i.imgur.com/oSGfc9k.png)
- Socket을 이용해 통신한다
- 사용하는 프로토콜은 TCP/IP이다
- 한 쪽은 Server / 다른 한 쪽은 Client
- 통신 방식은 Full-Duplex이다
- 두 PC는 네트워크로 연결되어 있다

즉, Socket은 **TCP/IP** 기반 네트워크 통신에서 데이터 송수신의 마지막 **접점**이다.

<br/>

### IP (Internet Protocol)
- 패킷 혹은 데이터그램이라고 하는 덩어리로 나뉘어 전송
- 주소와 전송 방식 지정
- 특징 : **비신뢰성, 비연결성**
- IPv4 : 현재 통용되는 주소 사용법 (32bit, 43억 개)
- IPv6 : 주소 확장 (128bit)

### TCP (Transmission Control Protocol)
- 전송 제어 프로토콜
- IP의 핵심 프로토콜 중 하나, IP 단점을 해결하자
- 연결 지향 (**3 way handshake**) : 3번 검증
    - SYN : 준비 됐니 ?
    - SYN-ACK : Yes
    - ACK : 받아라
- 특징
    - 데이터 전달 보증
    - 순서 보증
    - 신뢰할 수 있는 프로토콜
    - 느리다

### UDP (User Datagram Protocol)
- 데이터 전달, 순서 보장 X
- **속도가 빠름** (제약이 없기 때문에)
- 최신 HTTP에서는 UDP를 채택 (음성, 영상 전송)

<br/>

### 네트워크 원리
1. 노드끼리 연결 가까이 있다면, 직접 연결
- `A - 랜카드 - 랜선 - 랜카드 - B` 이런식으로
- **NIC (Network Interface Controller)** : 랜카드. 컴퓨터와 네트워크를 연결시키기 위한 카드로 대부분 메인보드에 내장되어 있음
2. **공유기**의 등장
- 연결 단자의 개수로부터 자유로움
- PC까리 연결하지 않고, 공유기를 거쳐 연결
- PC는 1개의 랜카드만 있으면 됨
3. 공유기끼리 연결
- 해저 케이블
- 인공위성
4. 랜선이라고 부르는 **이더넷 케이블**을 통해 데이터가 송수신됨
5. 네트워크 연결을 위해 **Kernel**의 도움을 받아 하드웨어로 접근이 가능함. 필요한건 **System Call**

### Port
동시 다발적 통신을 위한 가상의 문으로, Socket은 포트 내부에서 동작하는 부품이며, 포트 번호를 부여받음

<br/>

---

<br/>

## Socket 프로그래밍
![Server와 Client의 동작 방식](https://vos.line-scdn.net/landpress-content-v2_1761/1672029326745.png?updatedAt=1672029327000)

### TCP 기반 Server 소켓 동작 순서 [암기 !]
1. `socket()` 소켓 생성
2. `bind()` 소켓에 주소 할당
3. `listen()` 클라이언트 연결 요청 대기
4. `accept()` 클라이언트 연결 승인
5. `read()` / `write()` 통신
6. `close()` 소켓 닫기

### TCP 기반 Client 소켓 동작 순서 [암기 !]
1. `socket()` 소켓 생성
2. `connect()` 연결 요청
3. `read()` / `write()` 데이터 송수신
4. `close()` 연결 종료

<br/>

### Server와 Client의 차이
- **Server** : Client가 접속할 수 있도록 준비 (Port, IP 주소), 그리고 Client의 연결을 기다림
- **Client** : Server의 IP와 Port를 이용해 접속, 연결이 될 경우, 동작

|| 서버 | 클라이언트 |
|---|----|----|
| IP | 서버 소켓이 동작하는 **컴퓨터의 IP 주소** | 통신을 원하는 **원격지 PC의 IP 주소** |
| PORT | 소켓이 "위치 할" 포트 번호 | 소켓이 "위치한" 포트 번호 |

<br/>

### `echo_server.c`
```C
#include <stdio.h>          // 표준 입출력 함수를 사용하기 위한 헤더 파일
#include <sys/types.h>      // 데이터 타입 (socklen_t 등)을 사용하기 위한 헤더 파일
#include <sys/socket.h>     // 소켓 프로그래밍을 위한 헤더 파일
#include <signal.h>         // 시그널 처리를 위한 헤더 파일
#include <stdlib.h>         // 표준 라이브러리 함수 (exit 등)를 사용하기 위한 헤더 파일
#include <string.h>         // 문자열 처리 함수를 사용하기 위한 헤더 파일
#include <unistd.h>         // UNIX 표준 함수 (close, read, write 등)를 사용하기 위한 헤더 파일
#include <arpa/inet.h>      // 인터넷 주소 변환 함수 및 구조체 사용을 위한 헤더 파일

// 서버의 포트 번호 상수 선언
const char *PORT = "12345";

int server_sock;  // 서버 소켓 파일 디스크립터
int client_sock;  // 클라이언트 소켓 파일 디스크립터

// SIGINT (Ctrl + C) 시그널이 발생했을 때 호출되는 함수
void interrupt(int arg) {
    printf("\nYou typed Ctrl + C\n");  		// 시그널 수신 시 메시지 출력
    printf("Bye\n");

    close(client_sock);  			// 클라이언트 소켓 닫기
    close(server_sock);  			// 서버 소켓 닫기
    exit(1);             			// 프로그램 비정상 종료
}

// 문자열에서 개행 문자 '\n'을 제거하는 함수
void removeEnterChar(char *buf) {
    int len = strlen(buf); 			// 문자열 길이 계산
    for (int i = len - 1; i >= 0; i--) {  	// 문자열 끝에서부터 탐색
        if (buf[i] == '\n') {  			// 개행 문자를 찾으면
            buf[i] = '\0';  			// 널 문자로 대체하여 제거
            break;
        }
    }
}

int main() {
    // SIGINT 시그널을 수신하면 interrupt 함수를 실행하도록 설정
    signal(SIGINT, interrupt);

    // 서버 소켓 생성
    server_sock = socket(AF_INET, SOCK_STREAM, 0);
    if (server_sock == -1) {
        printf("ERROR :: 1_Socket Create Error\n");  	// 소켓 생성 실패 시 오류 메시지 출력
        exit(1);  					// 프로그램 비정상 종료
    }
    printf("Server On..\n");  				// 서버가 실행되었음을 알리는 메시지 출력

    // 소켓 옵션 설정: 이미 사용 중인 주소라도 재사용 가능하도록 설정
    int optval = 1;
    setsockopt(server_sock, SOL_SOCKET, SO_REUSEADDR, (void *)&optval, sizeof(optval));

    // 서버 주소 정보 설정
    struct sockaddr_in server_addr = {0};            // 서버 주소 구조체 초기화
    server_addr.sin_family = AF_INET;                // IPv4 프로토콜 사용
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY); // 모든 IP로부터의 연결을 허용
    server_addr.sin_port = htons(atoi(PORT));        // 서버 포트 번호 설정 (문자열을 정수로 변환 후 네트워크 바이트 순서로 변환)
    socklen_t server_addr_len = sizeof(server_addr); // 서버 주소 구조체의 크기

    // 소켓과 주소를 바인딩
    if (bind(server_sock, (struct sockaddr *)&server_addr, server_addr_len) == -1) {
        printf("ERROR :: 2_bind Error\n");  						// 바인딩 실패 시 오류 메시지 출력
        exit(1);  									// 프로그램 비정상 종료
    }
    printf("Bind Success\n");  // 바인딩 성공 메시지 출력

    // 클라이언트의 연결 요청을 대기 (백로그 큐 크기: 5)
    if (listen(server_sock, 5) == -1) {
        printf("ERROR :: 3_listen Error");  		// 연결 대기 실패 시 오류 메시지 출력
        exit(1);  					// 프로그램 비정상 종료
    }
    printf("Wait Client...\n");  			// 클라이언트 대기 메시지 출력

    client_sock = 0;  					// 클라이언트 소켓 초기화
    struct sockaddr_in client_addr = {0};  		// 클라이언트 주소 구조체 초기화
    socklen_t client_addr_len = sizeof(client_addr);  	// 클라이언트 주소 구조체의 크기

    // 서버 메인 루프: 클라이언트 연결 요청을 계속해서 수락
    while (1) {
        memset(&client_addr, 0, client_addr_len);  	// 클라이언트 주소 구조체 초기화

        // 클라이언트의 연결 요청을 수락
        client_sock = accept(server_sock, (struct sockaddr *)&client_addr, &client_addr_len);
        if (client_sock == -1) {
            printf("ERROR :: 4_accept Error\n");  						// 연결 수락 실패 시 오류 메시지 출력
            break;  										// 루프 탈출
        }
        printf("Client Connect Success!\n");  							// 클라이언트 연결 성공 메시지 출력

        char buf[100];  						// 송수신할 데이터 버퍼
        while (1) {
            memset(buf, 0, 100);  					// 버퍼 초기화

            int len = read(client_sock, buf, 99);  			// 클라이언트로부터 데이터 수신
            removeEnterChar(buf);  					// 수신한 데이터에서 개행 문자 제거

            if (len == 0) {  						// 클라이언트가 연결을 끊은 경우
                printf("INFO :: Disconnect with client... BYE\n");  	// 연결 종료 메시지 출력
                break;  						// 반복문 탈출
            }

            if (!strcmp("exit", buf)) {  				// 클라이언트가 "exit" 명령어를 보낸 경우
                printf("INFO :: Client want close... BYE\n");  		// 클라이언트의 종료 요청 메시지 출력
                break;  						// 반복문 탈출
            }
            write(client_sock, buf, strlen(buf));  			// 클라이언트로부터 받은 메시지를 그대로 에코하여 전송
        }
        close(client_sock);  						// 현재 클라이언트 소켓 닫기
        printf("Client Bye!\n");  					// 클라이언트 종료 메시지 출력
    }
    close(server_sock);  						// 서버 소켓 닫기
    printf("Server off..\n");  						// 서버 종료 메시지 출력 
    return 0;  								// 프로그램 정상 종료
}
```

<br/>

### `echo_client.c`
```C
#include <stdio.h>          // 표준 입출력 함수를 사용하기 위한 헤더 파일
#include <unistd.h>         // UNIX 표준 함수 (close, read, write 등)를 사용하기 위한 헤더 파일
#include <stdlib.h>         // 표준 라이브러리 함수 (exit, atoi 등)를 사용하기 위한 헤더 파일
#include <signal.h>         // 시그널 처리를 위한 헤더 파일
#include <sys/types.h>      // 데이터 타입 (socklen_t 등)을 사용하기 위한 헤더 파일
#include <sys/socket.h>     // 소켓 프로그래밍을 위한 헤더 파일
#include <netinet/in.h>     // 인터넷 프로토콜 관련 구조체 (sockaddr_in 등)를 사용하기 위한 헤더 파일
#include <arpa/inet.h>      // 인터넷 주소 변환 함수를 사용하기 위한 헤더 파일
#include <string.h>         // 문자열 처리 함수를 사용하기 위한 헤더 파일

// 서버의 IP 주소와 포트 번호 상수 선언
const char *SERVER_IP = "127.0.0.1";  
const char *SERVER_PORT = "12345";

int client_sock;  // 클라이언트 소켓 파일 디스크립터

// SIGINT (Ctrl + C) 시그널이 발생했을 때 호출되는 함수
void interrupt(int arg) {
    printf("\nYou typed Ctrl + C\n");           // 시그널 수신 시 메시지 출력
    printf("Bye\n");

    close(client_sock);                         // 클라이언트 소켓 닫기
    exit(1);                                    // 프로그램 비정상 종료
}

int main() {
    // SIGINT 시그널을 수신하면 interrupt 함수를 실행하도록 설정
    signal(SIGINT, interrupt);

    // 클라이언트 소켓 생성
    client_sock = socket(AF_INET, SOCK_STREAM, 0);
    if (client_sock == -1) {
        printf("ERROR :: 1_Socket Create Error\n");  	// 소켓 생성 실패 시 오류 메시지 출력
        exit(1);  					                    // 프로그램 비정상 종료
    }

    // 서버 주소 정보 설정
    struct sockaddr_in server_addr = {0};               // 서버 주소 구조체 초기화
    server_addr.sin_family = AF_INET;                   // IPv4 프로토콜 사용
    server_addr.sin_addr.s_addr = inet_addr(SERVER_IP); // 서버 IP 주소 설정
    server_addr.sin_port = htons(atoi(SERVER_PORT));    // 서버 포트 번호 설정 (문자열을 정수로 변환 후 네트워크 바이트 순서로 변환)
    
    socklen_t server_addr_len = sizeof(server_addr);    // 서버 주소 구조체의 크기

    // 서버에 연결 요청
    if (connect(client_sock, (struct sockaddr *)&server_addr, server_addr_len) == -1) {
        printf("ERROR :: 2_Connect Error\n");                                               // 연결 실패 시 오류 메시지 출력
        exit(1);                                                                            // 프로그램 비정상 종료
    }
    
    char buf[100];                                          // 송수신할 데이터 버퍼
    while (1) {
        memset(buf, 0, 100);                                // 버퍼 초기화
        scanf("%s", buf);                                   // 사용자 입력을 받아 버퍼에 저장
        if (!strcmp(buf, "exit")) {                         // 입력이 "exit"이면
            write(client_sock, buf, strlen(buf));           // 서버에 "exit" 메시지 전송
            break;                                          // 반복문 탈출
        }
        write(client_sock, buf, strlen(buf));               // 서버에 입력한 메시지 전송

        memset(buf, 0, 100);                                // 버퍼 초기화
        int len = read(client_sock, buf, 99);               // 서버로부터 응답 수신
        if (len == 0) {                                     // 서버가 연결을 끊은 경우
            printf("INFO :: Server Disconnected\n");        // 연결 종료 메시지 출력
            break;                                          // 반복문 탈출
        }
        printf("%s\n", buf);                                // 서버로부터 수신한 메시지 출력
    }

    close(client_sock);                                     // 클라이언트 소켓 닫기
    return 0;                                               // 프로그램 정상 종료
}
```

<br/>

### i. Server Socket 종료 시점
1. `socket()` 실패했을 때
```C
server_sock = socket(AF_INET, SOCK_STREAM, 0);
if (server_sock == -1){
    printf("ERROR :: 1_Socket Create Error\n");
    exit(1);
}
```

2. `bind()` 실패했을 때
```C
if (bind(server_sock, (struct sockaddr *)&server_addr, server_addr_len) == -1){
    printf("ERROR :: 2_bind Error\n");
    exit(1);
}
```

3. `listen()` 실패했을 때
```C
if (listen(server_sock, 5) == -1){
    printf("ERROR :: 3_listen Error");
    exit(1);
}
```

4. `accept()` 실패했을 때
```C
client_sock = accept(server_sock, (struct sockaddr *)&client_addr, &client_addr_len);
if (client_sock == -1) {
    printf("ERROR :: 4_accept Error\n"); 
    break;
}
```

5. Ctrl + C 누를 경우 
```C
void interrupt(int arg) {
    printf("\nYou typed Ctrl + C\n");
    printf("Bye\n");

    close(client_sock);  
    close(server_sock);  
    exit(1); 
}
```

### ii. Server의 Client Socket 종료 시점
1. Client의 접속이 끊어졌을 때
```C
int len = read(client_sock, buf, 99);
removeEnterChar(buf);

if (len == 0){
    printf("INFO :: Disconnect with client... BYE\n");
    break;
}
```

2. exit 입력을 받았을 때
```C
if (!strcmp("exit", buf)){
    printf("INFO :: Client want close ... BYE\n");
    break;
}
```

3. Ctrl + C 누를 경우
```C
void interrupt(int arg){
    close(client_sock);
    exit(1);
}
```

### iii. Client Socket 종료 시점
1. `socket()` 실패
```C
client_sock = socket(AF_INET, SOCK_STREAM, 0);
if (client_sock == -1) {
    printf("ERROR :: 1_Socket Create Error\n"); 
    exit(1); 
}
```

2. `connect()` 실패
```C
if (connect(client_sock, (struct sockaddr *)&server_addr, server_addr_len) == -1) {
    printf("ERROR :: 2_Connect Error\n");  
    exit(1);                                                                   
}
```

3. exit 입력했을 때
```C
if (!strcmp(buf, "exit")) {
    write(client_sock, buf, strlen(buf)); 
    break;  
}
```

4. Server가 종료되었을 때
```C
int len = read(client_sock, buf, 99); 
if (len == 0) {  
    printf("INFO :: Server Disconnected\n"); 
    break; 
}
```

5. Ctrl + C 누를 경우
```C
void interrupt(int arg) {
    printf("\nYou typed Ctrl + C\n"); 
    printf("Bye\n");

    close(client_sock);
    exit(1);
}
```

### NetCat(넷캣)
TCP/UDP 네트워크 연결을 통해 데이터는 읽거나 쓸 수 있도록 만든 유틸리티 프로그램

<br/>

### 네트워크 보안
- 가용성, 기밀성, 무결성에 대한 공격 및 장애로부터 모든 컴퓨터 리소스를 보호하는 것
- 다양한 공격이 있다 .. 컴퓨터 바이러스/해킹/랜섬웨어/악성코드/DDoS/트로이 목마/스푸핑, 스니핑 등등 ..
- 해결책
    - 개인: 암호 복잡도 강화, 보안 업데이트, 방화벽 및 안티바이러스 사용, VPN, 안전한 인터넷, 스팸 필터링, wifi 보안 등
    - 기업 : 보안 정책 수립, 방화벽 구성, IDS/IPS 도입, VPN 구성, 암호화, 보안 솔루션 업체에 위탁 등

### 제로 트러스트 모델
절대 신뢰하지 말고 항상 검증하라