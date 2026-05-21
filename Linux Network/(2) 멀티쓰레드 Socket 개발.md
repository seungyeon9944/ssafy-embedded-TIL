## Socket & Thread

Server에서 thread하는 이유:

**동시 접속한 client의 동작**을 처리하기 위해서 

### `chat_server.c`
```C
#include <stdio.h>             // 표준 입출력 함수
#include <signal.h>            // 신호 처리 함수
#include <unistd.h>            // 유닉스 표준 함수 정의
#include <stdlib.h>            // 표준 라이브러리 함수
#include <sys/types.h>         // 시스템 호출에 사용되는 데이터 타입
#include <sys/socket.h>        // 소켓 함수와 데이터 구조체
#include <arpa/inet.h>         // 인터넷 작업 관련 함수 (inet_addr 등)
#include <string.h>            // 문자열 처리 함수
#include <pthread.h>           // POSIX 스레드 라이브러리

#define MAX_CLIENT_CNT 500     // 최대 클라이언트 수

char PORT[6];                  // 서버 포트 번호를 저장할 전역 변수
int server_sock;               // 서버 소켓 디스크립터
int client_sock[MAX_CLIENT_CNT] = {0};  // 클라이언트 소켓 디스크립터 배열
struct sockaddr_in client_addr[MAX_CLIENT_CNT] = {0};  // 클라이언트 주소 구조체 배열

pthread_t tid[MAX_CLIENT_CNT]; // 클라이언트별 스레드 ID 배열
int exitFlag[MAX_CLIENT_CNT];  // 클라이언트별 종료 플래그 배열

pthread_mutex_t mlock;         // 임계 구역 보호를 위한 뮤텍스

// Ctrl + C 신호를 처리하는 함수
void interrupt(int arg){
    printf("\nYou typed Ctrl + C\n");
    printf("Bye\n");

    // 모든 클라이언트의 연결을 종료하고 스레드를 정리
    for (int i = 0; i < MAX_CLIENT_CNT; i++){
        if (client_sock[i] != 0){
            pthread_cancel(tid[i]);
            pthread_join(tid[i], 0);
            close(client_sock[i]);
        }
    }
    close(server_sock);  // 서버 소켓을 닫음
    exit(1);
}

// 문자열에서 '\n' 문자를 제거하는 함수
void removeEnterChar(char *buf){
    int len = strlen(buf);
    for (int i = len - 1; i >= 0; i--) {
        if (buf[i] == '\n') {
            buf[i] = '\0';
            break;
        }
    }
}

// 사용할 수 있는 클라이언트 ID를 반환하는 함수
int getClientID(){
    for (int i = 0; i < MAX_CLIENT_CNT; i++){
        if (client_sock[i] == 0)
            return i;
    }
    return -1;  // 사용 가능한 ID가 없을 경우 -1 반환
}

// 각 클라이언트를 처리하는 스레드 함수
void *client_handler(void *arg){
    int id = *(int *)arg;  // 클라이언트 ID를 받아옴

    char client_IP[100];  // 클라이언트 IP 주소를 저장할 변수
    strcpy(client_IP, inet_ntoa(client_addr[id].sin_addr));  // 클라이언트 IP 주소 복사
    printf("INFO :: Connect new Client (ID : %d, IP : %s)\n", id, client_IP);

    char buf[MAX_CLIENT_CNT]= { 0 };  // 메시지를 저장할 버퍼
    while (1) {
        memset(buf, 0, MAX_CLIENT_CNT);  // 버퍼 초기화
        int len = read(client_sock[id], buf, MAX_CLIENT_CNT);  // 클라이언트로부터 메시지 읽기
        if (len == 0) {  // 클라이언트가 연결을 끊었을 경우
            printf("INFO :: Disconnect with client.. BYE\n");
            exitFlag[id] = 1;  // 종료 플래그 설정
            break;
        }

        if (!strcmp("exit", buf)) {  // 클라이언트가 'exit' 메시지를 보낸 경우
            printf("INFO :: Client want close.. BYE\n");
            exitFlag[id] = 1;  // 종료 플래그 설정
            break;
        }

        removeEnterChar(buf);  // 메시지에서 '\n' 문자 제거
        pthread_mutex_lock(&mlock);  // 뮤텍스 잠금

        // 모든 클라이언트에게 메시지를 브로드캐스트
        for (int i = 0; i < MAX_CLIENT_CNT; i++){
            if (client_sock[i] != 0){
                write(client_sock[i], buf, strlen(buf));
            }
        }
        pthread_mutex_unlock(&mlock);  // 뮤텍스 해제
    }
    close(client_sock[id]);  // 클라이언트 소켓 닫기
}

int main(int argc, char* argv[]){
    if( argc<2 ){
        printf("ERROR Input Port Num\n");
        exit(1);
    }
    strcpy(PORT, argv[1]);  // 입력된 포트 번호 저장

    signal(SIGINT, interrupt);  // SIGINT 신호를 처리할 함수 설정
    pthread_mutex_init(&mlock, NULL);  // 뮤텍스 초기화

    server_sock = socket(AF_INET, SOCK_STREAM, 0);  // TCP 소켓 생성
    if (server_sock == -1){
        printf("ERROR :: 1_Socket Create Error\n");
        exit(1);
    }    
    printf("Server On..\n");    

    int optval = 1;
    setsockopt(server_sock, SOL_SOCKET, SO_REUSEADDR, (void *)&optval, sizeof(optval));  // 소켓 옵션 설정

    struct sockaddr_in server_addr = {0};  // 서버 주소 구조체 초기화
    server_addr.sin_family = AF_INET;  // 주소 체계를 IPv4로 설정
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);  // 모든 인터페이스에서 연결 허용
    server_addr.sin_port = htons(atoi(PORT));  // 포트 번호 설정

    if (bind(server_sock, (struct sockaddr *)&server_addr, sizeof(server_addr)) == -1){
        printf("ERROR :: 2_bind Error\n");
        exit(1);
    }
    printf("Bind Success\n");
    
    if (listen(server_sock, 5) == -1){  // 연결 대기열 설정
        printf("ERROR :: 3_listen Error");
        exit(1);
    }
    printf("Wait Client...\n");

    socklen_t client_addr_len = sizeof(struct sockaddr_in);
    int id_table[MAX_CLIENT_CNT];  // 클라이언트 ID 테이블

    while (1){
        int id = getClientID();  // 사용 가능한 클라이언트 ID 가져오기
        id_table[id] = id;

        if (id == -1){
            printf("WARNING :: Client FULL\n");  // 최대 클라이언트 수 초과
            sleep(1);
            continue;
        }

        memset(&client_addr[id], 0, sizeof(struct sockaddr_in));  // 클라이언트 주소 초기화

        // 클라이언트 연결 수락
        client_sock[id] = accept(server_sock, (struct sockaddr *)&client_addr[id], &client_addr_len);
        if (client_sock[id] == -1){
            printf("ERROR :: 4_accept Error\n");
            break;
        }

        // 클라이언트를 처리할 스레드 생성
        pthread_create(&tid[id], NULL, client_handler, (void *)&id_table[id]);

        // 종료된 클라이언트 스레드 정리
        for (int i = 0; i < MAX_CLIENT_CNT; i++){
            if (exitFlag[i] == 1){
                exitFlag[i] = 0;
                pthread_join(tid[i], 0);
                client_sock[i] = 0;
            }
        }
    }
    pthread_mutex_destroy(&mlock);  // 뮤텍스 제거
    close(server_sock);  // 서버 소켓 닫기
    return 0;
}
```

<br/>

그렇다면,

Client에서 thread하는 이유:

client에서 user가 메시지를 입력하는 도중에도,

server로부터 **다른 client가 입력한 메시지를 받아**오기 위해서

즉

1) 쉘을 통해 사용자로부터 메시지를 입력받는 Thread `receive_tid`와
2) Server로 메시지를 보내고, 출력하는 Thread `send_tid`

를 구분한다.

### `chat_client.c`
```C
#include <stdio.h>             // 표준 입출력 함수
#include <unistd.h>            // 유닉스 표준 함수 정의
#include <stdlib.h>            // 표준 라이브러리 함수
#include <signal.h>            // 신호 처리 함수
#include <sys/types.h>         // 시스템 호출에 사용되는 데이터 타입
#include <sys/socket.h>        // 소켓 함수와 데이터 구조체
#include <netinet/in.h>        // 인터넷 주소 체계 (sockaddr_in)
#include <arpa/inet.h>         // 인터넷 작업 관련 함수 (inet_addr 등)
#include <string.h>            // 문자열 처리 함수
#include <pthread.h>           // POSIX 스레드 라이브러리

#define NAME_SIZE 20           // 사용자 이름의 최대 크기
#define MSG_SIZE 100           // 메시지의 최대 크기

char name[NAME_SIZE];          // 사용자 이름을 저장할 전역 변수
char msg[MSG_SIZE];            // 메시지를 저장할 전역 변수

pthread_t send_tid;            // 메시지를 보내는 스레드의 ID
pthread_t receive_tid;         // 메시지를 받는 스레드의 ID
int exitFlag;                  // 프로그램 종료를 위한 플래그

char SERVER_IP[20];            // 서버 IP 주소를 저장할 전역 변수
char SERVER_PORT[6];           // 서버 포트 번호를 저장할 전역 변수

int client_sock;               // 클라이언트 소켓 디스크립터

// Ctrl + C 신호를 처리하는 함수
void interrupt(int arg){
    printf("\nYou typped Ctrl + C\n");
    printf("Bye\n");

    // 스레드를 취소하고 종료를 기다림
    pthread_cancel(send_tid);
    pthread_cancel(receive_tid);

    pthread_join(send_tid, 0);
    pthread_join(receive_tid, 0);

    // 소켓을 닫고 프로그램을 종료
    close(client_sock);
    exit(1);
}

// 메시지를 서버로 보내는 스레드 함수
void *sendMsg(){
    char buf[NAME_SIZE + MSG_SIZE + 1];  // 전송할 메시지를 저장할 버퍼

    while (!exitFlag){
        memset(buf, 0, NAME_SIZE + MSG_SIZE);  // 버퍼를 초기화
        
        fgets(msg, MSG_SIZE, stdin);  // 표준 입력으로부터 메시지를 입력받음
        if (!strcmp(msg, "exit\n")){  // 'exit' 입력 시 종료 플래그 설정
            exitFlag = 1;
            write(client_sock, msg, strlen(msg));  // 서버로 'exit' 메시지 전송
            break;
        }
        if (exitFlag) break;
        sprintf(buf, "%s %s", name, msg);  // 이름과 메시지를 합쳐 버퍼에 저장
        write(client_sock, buf, strlen(buf));  // 서버로 메시지 전송
    }
}

// 서버로부터 메시지를 받는 스레드 함수
void *receiveMsg(){
    char buf[NAME_SIZE + MSG_SIZE];  // 수신할 메시지를 저장할 버퍼
    while (!exitFlag){
        memset(buf, 0, NAME_SIZE + MSG_SIZE);  // 버퍼를 초기화
        int len = read(client_sock, buf, NAME_SIZE + MSG_SIZE - 1);  // 서버로부터 메시지 수신
        if (len == 0){  // 서버가 연결을 끊었을 경우
            printf("INFO :: Server Disconnected\n");
            kill(0, SIGINT);  // SIGINT 신호를 보내 프로그램을 종료
            exitFlag = 1;
            break;
        }
        printf("%s\n", buf);  // 수신한 메시지를 출력
    }
}

int main(int argc, char *argv[]){
    if( argc<4 ){
        printf("ERROR Input [IP Addr] [Port Num] [User Name]\n");
        exit(1);
    }
    strcpy(SERVER_IP, argv[1]);  // 서버 IP 주소 저장
    strcpy(SERVER_PORT, argv[2]);  // 서버 포트 번호 저장
    sprintf(name, "[%s]", argv[3]);  // 사용자 이름 저장

    signal(SIGINT, interrupt);  // SIGINT 신호를 처리할 함수 설정
    
    client_sock = socket(AF_INET, SOCK_STREAM, 0);  // TCP 소켓 생성
    if (client_sock == -1){
        printf("ERROR :: 1_Socket Create Error\n");
        exit(1);
    }
    //printf("Socket Create!\n");
    
    struct sockaddr_in server_addr = {0};  // 서버 주소 구조체 초기화
    server_addr.sin_family = AF_INET;  // 주소 체계를 IPv4로 설정
    server_addr.sin_addr.s_addr = inet_addr(SERVER_IP);  // 서버 IP 주소 설정
    server_addr.sin_port = htons(atoi(SERVER_PORT));  // 서버 포트 번호 설정
    socklen_t server_addr_len = sizeof(server_addr);
    
    // 서버에 연결 시도
    if (connect(client_sock, (struct sockaddr *)&server_addr, server_addr_len) == -1){
        printf("ERROR :: 2_Connect Error\n");
        exit(1);
    }
    //printf("Connect Success!\n");
    
    // 메시지를 보내고 받는 스레드 생성
    pthread_create(&send_tid, NULL, sendMsg, NULL);
    pthread_create(&receive_tid, NULL, receiveMsg, NULL);

    // 스레드가 종료될 때까지 대기
    pthread_join(send_tid, 0);
    pthread_join(receive_tid, 0);

    // 소켓을 닫고 프로그램 종료
    close(client_sock);
    return 0;
}
```
- 각각의 Thread는 종료되면 **자원 정리를 위해 `join` 필수**
    ```C
    pthread_create(&send_tid, NULL, sendMsg, NULL);
    pthread_create(&receive_tid, NULL, receiveMsg, NULL);

    pthread_join(send_tid, 0);
    pthread_join(receive_tid, 0);
    ```
- `fgets()`를 이용해 `scanf()`로 생기는 에러 방지 (입력 대기 동안 다른 사람의 메시지 출력 안됨)
- `exit` 입력 시 Thread 종료
    - `exitFlag = 1` → `sendMsg()` 종료
    - 서버에도 메시지 전송
- `kill(0, SIGINT)`
    - SIGINT 시그널 전송
    - 0 → 현재 프로세스에 속한 모든 스레드 및 프로세스

<br/>

---

<br/>

## OSI 7 Layer

ISO에서 정한, 통신의 추상화 단계

![OSI 7계층](https://i.imgur.com/b8qVx3o.png)

### 계층화를 통한 장점
**커뮤니케이션 용도**
- **HW 장비 지칭 및 커뮤니케이션**
    - 통신에서 어떤 역할을 하는 장비인지 Layer 번호로 표현
    - 장비관리 수월
    - 추상화로 Interface만 맞추면 다른 장비들 개발 가능
- **SW 소스코드 및 커뮤니케이션**
    - 통신에서 어떤 역할을 하는 소스코드인지 Layer 번호로 표현
    - 추상화로 Interface를 나눠서 개발
    - 유지보수 관리
- Data : **어떤 단계에서 만들어진 Data인지 Layer 번호로 표현** 
    - App에서 전달하고 싶은 메시지가 있음
    - 각 단계별로 **헤더** 정보를 하나씩 추가
    - 최종적으로 만들어진 데이터를 **프레임**이라고 함
    - 마지막 레이어에서 데이터가 전송됨

<br/>

### L7 - Application Layer (응용 계층)
- App에서 Data가 만들어지는 구간
- L6, L5의 동작이 포함됨
    - L6 : 표현 계층, 데이터 압축, 암호화 정보 포함
    - L5 : 세션 계층, 통신 동기화 정보 포함
- ex) 웹 사이트 접속
    - 웹서버에 요청 시작
    - HTTP 프로토콜 사용 등등의 메시지 생성 (데이터)
- L7 스위치
    - URL / 쿠키 등을 확인하여 섬세한 로드밸런싱 가능
    - 컨텐츠 내용을 확인 후 스위칭 가능, 보안도 담당
    - **로드밸런싱** (Load Balancing) : 부하 분산 - 트래픽을 감지하고 서버의 부하를 줄여주는 것


### L4 - Transport Layer (전송 계층)
- 이 곳에서 TCP Header가 추가됨 (출발지 **포트번호**, 목적지 **포트번호** 포함)
- TCP 헤더를 가진 데이터, **세그먼트**라고 함 
- L4 스위치
    - 로드 밸런싱
    - 헤더 정보 분석해서 TCP/포트 정보 분석해서 패킷 처리


### L3 - Network Layer (네트워크 계층)
- 이 곳에서 IP Header가 추가됨 (출발지 **IP** 주소, 목적지 **IP** 주소)
- IP 헤더를 가진 데이터, **패킷**이라고 함
- 라우터 (SW 기반)
    - 네트워크 데이터 전달 (IP 확인)
    - L3 스위치 (라우터 기능을 스위치에 내장, HW 기반, 좀 더 쌈)


### L2 - Data Link Layer (데이터 링크 계층)
- 이 곳에서 이더넷 Header가 추가됨 (출발지 **MAC** 주소, 목적지 **MAC** 주소)
- 이더넷 헤더를 가진 데이터, **이더넷 프레임**이라고 함
- 스위치
    - LAN 영역의 Data 정보전달 (Ethernet / MAC 연관)

### L1 - Physical Layer (물리 계층)
- 데이터를 전기 신호로 변환
- H/W 장비를 통해 전송되거나 전송받음
- 전기적 신호를 처리하는 곳
    - 허브 (신호 복사 역할)
    - NIC : Network Interface Controller
    - UTP 케이블 

### MAC 주소
기기마다 갖고 있는 고유 번호
- 네트워크마다 기기를 구분하는 주소
- 48bit 길이의 숫자 조합으로 구성, 일반적으로 16진수 표현
- 제조업체에서 NIC에 물리적으로 부여

<br/>

---

<br/>

## 네트워크

### LAN
- Local Area Network
- 랜카드 : 근거리 통신망을 위한 네트워크 카드
- 랜케이블 : 근거리 통신망용 케이블
- 집안 / 사무실 / 소규모 공장 수준

### WAN
- Wild Area Network
- LAN보다 넓은 네트워크
- 기업 지사끼리 연결된 네트워크, 대규모 공장, 군대 (인트라넷)

네트워크 규모로 보면 Internet > WAN > LAN

### Cell (셀)
무선 이동통신을 위해 하나의 기지국이 포괄하는 범위

### Cellular (셀룰러)
한 지역을 여러 개의 셀로 분할하여 운용하는 통신 방식

<br/>

---

<br/>

## Ethernet

### LAN 통신의 표준
- **CSMA / CD 프로토콜**을 쓰고 (MAC 주소 기반 통신)
- 이더넷 카드 장비가 있어야 하고
- 이더넷 케이블을 사용하는 통신 방법

### LAN에서 통신하는 방식
모든 노드에게 메시지를 보낸다.

### CSMA / CD 프로토콜 동작 방식
1. 현재 네트워크에 통신이 되고 있는지 Check (캐리어 신호 유무)
2. 사용 가능할 때까지 랜덤 시간 기다림
3. 통신이 안 되고 있으면 통신을 시작
4. 만약 충돌이 발생한다면, 랜덤 시간 이후 재전송

### Collision Domain
충돌이 발생할 수 있는 네트워크 범위
- LAN 네트워크는 모두 같은 콜리젼 도메인을 가짐
- 연결되는 기기가 많아질수록 콜리젼 확률이 올라가 통신 효율이 떨어질 수 있음
- 네트워크 장비를 사용하여 콜리젼 도메인 축소하여 원활한 통신을 가능하게 하자 !

해결방법
1. **스위치 사용** : 스위치를 사용해 개별 포트 간에 트래픽 분리
2. **VLAN 사용** : 가상 랜을 사용해 네트워크를 분리하고 네트워크 성능 향상과 보안성 높임
3. 네트워크 장비의 **포트 제한**
4. **네트워크 분할**
5. 네트워크 장비에서의 **전송률 제한**

<br/>

### 네트워크 구성
- ISP 업체에 전용 회선과 IP 할당
- 라우터 연결
    - 게이트웨이 역할 : 외부 네트워크끼리 연결에 사용
    - 외부 네트워크 : IP 기반 통신 (TCP/IP)
    - 내부 네트워크 : MAC 기반 (CSMA/CD)
- 스위치
    - MAC 기반으로 통신 데이터 전달

![네트워크 구성](https://i.imgur.com/I2zKFtL.png)