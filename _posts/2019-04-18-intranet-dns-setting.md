---
title: "사내 네임서버 운영하기"
date: 2019-04-18 08:26:28 -0400
categories: server
---

네임서버 운영 매뉴얼

클라이언트 DNS 주소 변경

아래를 배치파일로 만들어 관리자로 실행
`netsh interface ip set dns "이더넷" static 192.168.124.200`
`netsh interface ip show config`
`SET /P P=아무키나 누르세요...`


서버 세팅

1. IP 할당 -> 가상 네트워크 드라이브 추가
우분투 콘솔 이용 할당

2. 호스트 네임 설정
`sudo nano /etc/hosts`

[요청IP]		주소명
`ex)192.168.124.202	file.ycs.com`

`sudo nano /etc/hostnames`

[호스트명]
`ex)file.ycs.com`


3. Bind9 이용 주소 매핑

  0) 설치 
`sudo apt-get install -y Bind9`

  1)zone 추가

`sudo nano /etc/bind/named.conf.local`

`zone "124.168.192.in-addr.arpa" {type master; file "/etc/bind/db.124.168.192";};//PTR 존 추가`
`zone "ycs.com" {type master; file "/etc/bind/db.ycs.com";}; //ns 존 추가`


  2)NS 설정

`sudo nano /etc/bind/db.ycs.com  //IP주소 마지막 클래스 자리를 제외한 나머지 역순으로`

```$TTL 1 // DNS 캐시 리프레시 주기 초단위 
@       IN      SOA     ycs.com. root.ycs.com. (
                              3         ; Serial	//설정 바로 적용을 위해 변경시 시리얼 숫자 +1 또는 그냥 TTL 주기를 짧게
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL


도메인				주소
www.ycs.com.    IN      A       192.168.124.200
git.ycs.com.	IN	A	192.168.124.201
```
  3)PTR 설정
`sudo nano /etc/bind/db.124.168.192  //IP주소 마지막 클래스 자리를 제외한 나머지 역순으로`
PTR 레코드에 주소 추가

`200 IN PTR ycs.com.  //뒤에 . 에 유의
201 IN PTR git.ycs.com.`

  4)설정적용
`sudo service bind9 restart`

4. iptables 이용 포트 Redirect

1) 포트 설정
`sudo iptables -t nat -I PREROUTING --src 0/0 --dst [요청IP] -p tcp --dport [요청 포트] -j REDIRECT --to-ports [보낼 포트]`
`ex) sudo iptables -t nat -I PREROUTING --src 0/0 --dst 192.168.124.202 -p tcp --dport 80 -j REDIRECT --to-ports 8080`

5. 설정 저장 (안하면 껏다키면 설정 사라짐)
1) iptables-persistent 설치
`sudo apt-get install -y iptables-persistent`
2) 저장
`sudo iptables-save`

* 설정 되돌리기
1) 설정 목록확인
`sudo iptables -t nat --line-numbers -L`

2) 해당 설정의 번호(num column) 항목 보고 지우기
`sudo iptables -t nat -D PREROUTING [번호]`
`ex) sudo iptables -t nat -D PREROUTING 5`
