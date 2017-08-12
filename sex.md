# Payload Write-up
## Binary Reversing

바이너리를 IDA로 열어보면 C++로 작성되어있는 menu challenge란 걸 알 수 있다. 
맨 처음에 인자로 3개의 파일을 입력받고 3개의 네트워크 디바이스 구조체를 할당해준 후 실제 동작 부분으로 넘어간다. 

1번 메뉴를 입력하면 새로운 네트워크 디바이스를 설정해 줄 수 있고, 2번 메뉴를 입력하면 네트워크 디바이스를 free한다. 그리고 3번 메뉴를 입력하면 thread를 생성하고 그 함수에 파일의 내용과 네트워크 디바이스 객체를 인자로 전달해준다. 이런 개략적인 흐름을 안 상태에서 가장 먼저 해야 하는 것은 사용자 입력의 추적이다.

3번 메뉴의 threadFunc에서 filepath를 인자로 주고 함수 내부에서 open()으로 읽고 파일내용을 읽어온 후 파일내용을 파싱해 type에 일치하는 처리를 해준다.

네트워크 디바이스 객체는 ethernet layer, arp layer, ip layer, tcp layer 총 4가지 계층으로 구성돼있으며 실제 threadFunc에서는 파일내용을 파싱해 헤더 구조에 알맞는 처리를 해준다. 즉, 이 바이너리는 pcap 파일을 파싱해오는 바이너리라 할 수 있겠다. 실제 코드를 보면 Ethernet, ARP, IP, TCP의 Header을 구현해 파일에서 파싱해온 부분과 비교해 해당하는 vtable을 호출하는 식으로 처리한다.

* threadFunc
	파일에 저장된 값 읽어온 후 파싱 시작.

	* ARPrecv()
		파싱해온 결과값에 따라 Data, ARP Request, ARP Reply로 구분한 후 처리한다.

	* IPrecv()
		Destination IP Address와 파일에서 파싱해온 값을 비교해서 일치할 경우 tcpRecv() 함수를 호출한다.
		* tcpRecv()
			Acknowledgement Number 값에 따라 값을 처리한다. 520일 경우 새로운 구조체를 할당해 +192 부분에 저장, 514일 경우 +192 부분에 저장된 구조체를 free, 512일 경우 +192에 저장된 구조체에 접근해 함수테이블 포인터를 호출한다.

        [ Function Diagram 1-1 ]

## Vulnerability

tcpRecv() 함수에서 Acknowledgement Number가 512일 때 함수테이블 포인터를 호출하는데 여기서 접근할 때 포인터의 무결성에 대해 어떠한 검사도 하지 않는다. 즉, 맨 처음에 number 520으로 할당, file을 바꿔서 number 514로 해제, ARPrecv() 함수 안에서 Data를 넣어 포인터 테이블을 덮어쓴 후 파일을 다시 number 512로 바꿔서 우리가 원하는 곳으로 프로그램의 Control Flow를 바꿔줄 수 있다.