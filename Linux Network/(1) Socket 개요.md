## Socket 통신
Socket을 이용해, **서버-클라이언트** 간 데이터를 주고받는 양방향 연결 지향성 통신

![Socket 통신](https://i.imgur.com/iBCStLL.png)
- Socket을 이용해 통신한다
- 사용하는 프로토콜은 TCP/IP이다
- 한 쪽은 Server / 다른 한 쪽은 Client
- 통신 방식은 Full-Duplex이다
- 두 PC는 네트워크로 연결되어 있다

즉, Socket은 **TCP/IP** 기반 네트워크 통신에서 데이터 송수신의 마지막 **접점**이다.

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

---

## Socket 프로그래밍
![Server와 Client의 동작 방식](https://vos.line-scdn.net/landpress-content-v2_1761/1672029326745.png?updatedAt=1672029327000)

### TCP 기반 Server 소켓 동작 순서 [암기 !]
1. `socket()` 소켓 생성
2. `bind()` 소켓에 주소 할당
3. `listen()` 클라이언트 연결 요청 대기
4. `accept()` 클라이언트 연결 승인
5. `read()` / `write()` 통신
6. `close()` 소켓 닫기