# Cisco Multicasting on AWS  
AWS 환경에서 Cisco Multicasting을 구축하는 표준화된 절차와 명령어를 정리한 가이드입니다.  
가독성과 유지보수를 위해 단계별로 구성하였으며, 각 단계별 주요 명령어와 참고 링크를 포함합니다.

---

## 목차

- [1. VPC 생성](#1-vpc-생성)
- [2. Security Group 생성](#2-security-group-생성)
- [3. EIP 및 Network Interface 생성](#3-eip-및-network-interface-생성)
- [4. Multicast Source/Member 인스턴스 준비](#4-multicast-sourcemember-인스턴스-준비)
- [5. Transit Gateway 구성](#5-transit-gateway-구성)
- [6. VR 생성 및 설정](#6-vr-생성-및-설정)
- [7. VR 접속 및 기본 설정](#7-vr-접속-및-기본-설정)
- [8. ge2/ge3 인터페이스 구성](#8-ge2ge3-인터페이스-구성)
- [9. VR의 Default Route 변경](#9-vr의-default-route-변경)
- [10. VR(ge2) 접속 및 외부 연결 확인](#10-vrge2-접속-및-외부-연결-확인)
- [11. Tunnel 인터페이스 생성](#11-tunnel-인터페이스-생성)
- [12. IP Multicast Routing 구성](#12-ip-multicast-routing-구성)
- [13. PIM 구성](#13-pim-구성)
- [14. PIM RP 구성](#14-pim-rp-구성)
- [15. IGMP Static Group 구성](#15-igmp-static-group-구성)
- [16. (선택) Secondary IP Address 구성](#16-선택-secondary-ip-address-구성)
- [17. (선택) IGMPv2 참고](#17-선택-igmpv2-참고)
- [참고 자료](#참고-자료)

---

## 1. VPC 생성
- AWS 콘솔 또는 CLI를 통해 VPC를 생성합니다.

## 2. Security Group 생성
- 멀티캐스트 트래픽 및 SSH 등 필요한 포트를 허용하는 보안 그룹을 생성합니다.

## 3. EIP 및 Network Interface 생성
- Elastic IP 및 Network Interface를 생성하여 인스턴스에 할당합니다.

## 4. Multicast Source/Member 인스턴스 준비

**iperf 설치**
```bash
sudo amazon-linux-extras install epel -y
sudo yum-config-manager --enable epel
sudo yum search iperf
sudo su - root
yum install iperf.x86_64 -y
```

**iperf Client 실행**
```bash
iperf -c 224.9.9.5 -u -T 10 -t 10000 -i 1 -b 1024 -p 5001 -l 300
```

**iperf Server 실행**
```bash
iperf -s -i 1 -B 224.9.9.5 -u -p 5001
```

## 5. Transit Gateway 구성

- Transit Gateway 생성
- Attachments 및 Multicast 도메인 설정
  - 도메인 생성
  - Association 생성
  - Group IP Address에 source/member 지정

## 6. VR 생성 및 설정

- **Cisco CSR 1000V**: AWS Marketplace에서 이미지 선택  
- **Nitro 인스턴스 타입**: AWS 공식 문서 참고
- **VR-AWS, VR-IDC** 인스턴스 생성 및 source/dest. check 해제 (ge2/ge3)

## 7. VR 접속 및 기본 설정

```bash
ssh -i lz-capital-singapore.pem ec2-user@10.101.0.195
ssh -i lz-capital-singapore.pem ec2-user@10.201.0.138
```
```shell
conf t
hostname VR-AWS
exit

conf t
hostname VR-IDC
exit
```

## 8. ge2/ge3 인터페이스 구성

**인터페이스 상태 및 주소 확인**
```shell
sho ip int br
sh int gigabitEthernet 2
sh int gigabitEthernet 3
sho conf
```

**인터페이스 IP 설정 예시**
```shell
# AWS-VR ge2
conf t
interface gigabitEthernet 2
ip address 10.101.1.107 255.255.255.0
no shutdown
end
wr

# AWS-VR ge3
conf t
interface gigabitEthernet 3
ip address 10.101.2.112 255.255.255.0
no shutdown
end
wr

# IDC-VR ge2
conf t
interface gigabitEthernet 2
ip address 10.201.1.136 255.255.255.0
no shutdown
end
wr

# IDC-VR ge3
conf t
interface gigabitEthernet 3
ip address 10.201.2.207 255.255.255.0
no shutdown
end
wr
```

**연결 확인**
```shell
ping 10.101.1.1
ping 10.101.2.1
ping 10.201.1.1
ping 10.201.2.1
```

## 9. VR의 Default Route 변경

**라우팅 테이블 확인**
```shell
sho ip route
sho conf
```

**Default Route 변경**
```shell
# VR-AWS
conf t
ip route 0.0.0.0 0.0.0.0 10.101.1.1
no ip route 0.0.0.0 0.0.0.0 GigabitEthernet1 10.101.0.1

# VR-IDC
conf t
ip route 0.0.0.0 0.0.0.0 10.201.1.1
no ip route 0.0.0.0 0.0.0.0 GigabitEthernet1 10.201.0.1
```

## 10. VR(ge2) 접속 및 외부 연결 확인

```bash
ssh -i lz-capital-singapore.pem ec2-user@10.101.1.107
ssh -i lz-capital-singapore.pem ec2-user@10.201.1.136
```
```shell
ping 8.8.8.8
```

## 11. Tunnel 인터페이스 생성

**VR-AWS**
```shell
conf t
interface Tunnel0
ip address 192.168.1.1 255.255.255.0
tunnel source gigabitEthernet 2
tunnel destination 18.143.222.1
end
wr
```

**VR-IDC**
```shell
conf t
interface Tunnel0
ip address 192.168.1.2 255.255.255.0
tunnel source gigabitEthernet 2
tunnel destination 52.221.91.157
end
wr
```

**연결 확인**
```shell
ping 192.168.1.1
ping 192.168.1.2
ping 18.143.222.1
ping 52.221.91.157
```

## 12. IP Multicast Routing 구성

```shell
conf t
ip multicast-routing distributed
```

## 13. PIM 구성

```shell
conf t
interface Tunnel0
ip pim sparse-mode
end

conf t
interface gigabitEthernet 3
ip pim sparse-mode
end
wr

sho ip pim neighbor
```

## 14. PIM RP 구성

```shell
conf t
ip pim rp-address 192.168.1.2
end
```

## 15. IGMP Static Group 구성 (VR-AWS)

```shell
conf t
interface gigabitEthernet 3
ip igmp static-group 224.9.9.5
exit
end

show ip mroute
```

## 16. (선택) Secondary IP Address 구성 (VR-IDC)
**Multicast 소스 네트워크에 대한 RPF(Reverse Path Forwarding) 문제를 해결하기 위한 방법**
```shell
conf t
interface gigabitEthernet 3
ip address 10.201.3.76 255.255.255.0 secondary
```
```shell
sho ip mroute
sho int summary
```

## 17. (선택) IGMPv2 참고

- AWS Transit Gateway 멀티캐스트 관련 고려사항 및 IGMPv2 지원 여부는 [AWS 공식 문서](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-multicast-overview.html) 참고

---

## 참고 자료

- [Cisco Multicast over GRE](https://www.cisco.com/c/en/us/support/docs/ip/ip-multicast/43584-mcast-over-gre.html)
- [AWS Transit Gateway 멀티캐스트 공식 가이드](https://docs.aws.amazon.com/ko_kr/vpc/latest/tgw/tgw-multicast-overview.html)
- [AWS Marketplace: Cisco CSR 1000V](https://us-east-1.console.aws.amazon.com/marketplace/home#/search!mpSearch/search?text=Cisco+Cloud+Services+Router+%28CSR%29+1000V)
- [AWS EC2 Nitro 인스턴스 타입](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-types.html#ec2-nitro-instances)
- [AWS 기술 블로그: AWS를 활용한 확장성 높은 모바일 트레이딩 시스템 (MTS) 구축하기](https://aws.amazon.com/ko/blogs/tech/aws-mts-scalability-mobile-trading-system)
