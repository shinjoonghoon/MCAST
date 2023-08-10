# [Cisco Multicasting]

# 1. VPC 생성
# 2. Security Group 생성
# 3. EIP와 Network Interfaces 생성
# 4. Multicast source/member instance 생성
* iperf 설치
```
sudo amazon-linux-extras install epel -y
sudo yum-config-manager --enable epel
sudo yum search iperf
sudo su - root
yum install iperf.x86_64 -y
```
* iperf client
```
iperf -c 224.9.9.9 -u -T 10 -t 10000 -i 1 -b 1024 -p 5001 -l 300
```
* iperf server
```
iperf -s -i 1 -B 224.9.9.9 -u -p 5001
```
# 5. Transit Gateway 구성
* Transit Gateway 생성
* Transit Gateway Attachments
* Transit Gateway Multicast
  - Domain 생성
  - Association 생성
  - Group IP Address에 해당하는 source/memeber 구성
# 6. VR 생성
* Cisco CSR 1000V
```
https://us-east-1.console.aws.amazon.com/marketplace/home#/search!mpSearch/search?text=Cisco+Cloud+Services+Router+%28CSR%29+1000V
```
* Nitro type
```
https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-types.html#ec2-nitro-instances
```
* VR-AWS
* VR-IDC
* source/dest. check 해제
  - ge2/ge3에 대해서
# 7. VR(ge1) 접속
```
ssh -i lz-capital-singapore.pem ec2-user@10.101.0.195
```
```
ssh -i lz-capital-singapore.pem ec2-user@10.201.0.138
```
```
conf t
hostname VR-AWS
exit
```
```
conf t
hostname VR-IDC
exit
```
# 8. ge2와 ge3 구성
```
sho ip int br
```
* ge2/ge2 NiC address 확인
```
sh int gigabitEthernet 2
```
```
sh int gigabitEthernet 3
```
```
sho ip int br
sho conf
```
```
#AWS-VR ge 2
configure t
interface gigabitEthernet 2
ip address 10.101.1.107 255.255.255.0
no shutdown
end
wr

#AWS-VR ge 3
configure t
interface gigabitEthernet 3
ip address 10.101.2.112 255.255.255.0
no shutdown
end
wr

#IDC-VR ge 2
configure t
interface gigabitEthernet 2
ip address 10.201.1.136 255.255.255.0
no shutdown
end
wr

#IDC-VR ge 3
configure t
interface gigabitEthernet 3
ip address 10.201.2.207 255.255.255.0
no shutdown
end
wr
```
```
sho ip int br
```
* validation
```
ping 10.101.1.1
ping 10.101.2.1
ping 10.201.1.1
ping 10.201.2.1
```
# 9. VR의 default routing 변경
```
sho ip route
```
```
sho conf
```
* VR-AWs
```
conf t
ip route 0.0.0.0 0.0.0.0 10.101.1.1
no ip route 0.0.0.0 0.0.0.0 GigabitEthernet1 10.101.0.1
```
* VR-IDC
```
conf t
ip route 0.0.0.0 0.0.0.0 10.201.1.1
no ip route 0.0.0.0 0.0.0.0 GigabitEthernet1 10.201.0.1
```
```
sho ip route
```
# 10. VR(ge2) 접속
```
ssh -i lz-capital-singapore.pem ec2-user@10.101.1.107
ssh -i lz-capital-singapore.pem ec2-user@10.201.1.136
```
* validation
```
ping 8.8.8.8
```

# 11. interface Tunnel 생성
* VR-AWS
```
config t
interface Tunnel0
ip address 192.168.1.1 255.255.255.0
tunnel source gigabitEthernet 2
tunnel destination 18.143.222.1
end
wr
```
* VR-IDC
```
config t
interface Tunnel0
ip address 192.168.1.2 255.255.255.0
tunnel source gigabitEthernet 2
tunnel destination 52.221.91.157
end
wr
```
```
sho ip int br
```
* validation
```
ping 192.168.1.1
ping 192.168.1.2
```
```
ping 18.143.222.1
ping 52.221.91.157
```
# 12. IP Multicast routing 구성
```
conf t
ip multicast-routing distributed
```
# 13. Tunnel0와 ge3에 PIM 구성
```
config t
interface Tunnel0
ip pim sparse-mode
end

configure t
interface gigabitEthernet 3
ip pim sparse-mode
end
wr

sho ip pim neighbor
```
# 14. PIM RP 구성
```
ip pim rp-address 192.168.1.2
```
# 15. VR-AWS에서 igmp static-group 구성
```
config t
interface gigabitEthernet 3
ip igmp static-group 224.9.9.9
exit
end
```
```
show ip mroute
```
# 16. (Optional) VR-IDC(ge3)에 secondary ip address 구성
```
conf t
interface gigabitEthernet 3
ip address 10.201.3.76 255.255.255.0 secondary
```
```
sho ip mroute
```
```
sho int summary
```
# 17. (Optional) IGMPv2
* Multicast on transit gateways - Considerations: https://docs.aws.amazon.com/vpc/latest/tgw/tgw-multicast-overview.html

# ref
https://www.cisco.com/c/en/us/support/docs/ip/ip-multicast/43584-mcast-over-gre.html
  
