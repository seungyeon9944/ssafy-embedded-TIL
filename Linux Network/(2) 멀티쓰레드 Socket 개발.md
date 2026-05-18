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

1) 쉘을 통해 사용자로부터 메시지를 입력받는 Thread와
2) Server로 메시지를 보내고, 출력하는 Thread

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

---

## OSI 7 Layer

---

## 네트워크

-- 

## Ethernet