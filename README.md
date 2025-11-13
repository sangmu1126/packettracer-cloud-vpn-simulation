# 🚀 10-Router Enterprise WAN Integration Project (OSPF/EIGRP/HSRP/NAT)

## 🎯 프로젝트 개요 (Project Goal)

본 프로젝트는 OSPF 멀티 에어리어(Multi-Area)와 EIGRP 자율 시스템(AS)을 통합하고, NAT를 통한 인터넷 게이트웨이 및 HSRP를 통한 게이트웨이 이중화까지 구현한 **복합 라우팅 기반의 기업 내부망 구축** 프로젝트입니다.

### 핵심 기술 검증

| 기술 | 역할 | 구현 장비 |
| :--- | :--- | :--- |
| **OSPF Multi-Area** | 백본(Area 0)과 지사(Area 1) 분리 | R0, R1, R2, R4, R6, R8 |
| **EIGRP Redistribution**| 이질적인 프로토콜 영역 통합 | **R3 (ASBR)** |
| **HSRP** | 게이트웨이 이중화 및 로드 밸런싱 | R6, R8 $\leftrightarrow$ Switch12 |
| **NAT Overload (PAT)** | 내부망의 외부 인터넷 접속 | **R4 (Internet GW)** |

-----

## 🗺️ 1. 최종 아키텍처 및 IP 할당

| 라우터/장비 | 역할 (Role) | OSPF/EIGRP 영역 | 핵심 IP 대역 |
| :---: | :--- | :--- | :--- |
| **R4** | **Internet GW / NAT** | OSPF Area 0 | `200.0.0.2 /30` (Public) |
| **R3** | **ASBR** | OSPF 0 & EIGRP 100 | `10.0.0.3` (Core) / `10.3.0.3` (EIGRP Hub) |
| **R2** | **ABR** | OSPF 0 & OSPF 1 | `10.2.0.2` (Area 1 Hub) |
| **R8 & R6** | **HSRP Gateway** | OSPF Area 1 | **VIP:** `192.168.8.1` / `192.168.81.1` |
| **R5, R7, R9** | **Branch (EIGRP)** | EIGRP 100 | `192.168.5.x`, `192.168.7.x`, `192.168.9.x` |
| **ISP\_Router** | Internet Simulation | Static Route | `8.8.8.8` (Loopback) |

-----

## 2\. 📜 주요 라우터 설정 파일 (Critical Configuration Files)

다음은 라우팅 경계 및 핵심 기능을 수행하는 라우터들의 최종 설정입니다.

### 2.1. Router4 (Internet GW / NAT / OSPF Originator)

```cisco
hostname Router4
enable secret class
no ip domain-lookup
ip routing
!
interface GigabitEthernet0/0
 ip address 10.0.1.4 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
interface GigabitEthernet0/1
 ip address 10.0.2.4 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
interface GigabitEthernet0/2.4
 encapsulation dot1Q 4
 ip address 192.168.4.1 255.255.255.0
 ip nat inside
 no shutdown
!
interface GigabitEthernet0/2.200
 encapsulation dot1Q 200
 ip address 200.0.0.2 255.255.255.252
 ip nat outside
 no shutdown
!
router ospf 1
 network 192.168.4.0 0.0.0.255 area 0
 default-information originate
!
ip route 0.0.0.0 0.0.0.0 200.0.0.1
!
ip access-list extended ACL_FOR_NAT
 permit ip 192.168.0.0 0.0.255.255 any
 permit ip 10.0.0.0 0.255.255.255 any
!
ip nat inside source list ACL_FOR_NAT interface GigabitEthernet0/2.200 overload
!
end
```

### 3.2. Router3 (ASBR: OSPF $\leftrightarrow$ EIGRP Redistribution)

```cisco
hostname Router3
enable secret class
no ip domain-lookup
ip routing
!
interface GigabitEthernet0/0
 ip address 10.0.0.3 255.255.255.0
 ip ospf cost 1
 ip ospf 1 area 0
 no shutdown
!
interface GigabitEthernet0/1
 ip address 10.0.4.3 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
interface GigabitEthernet0/2
 ip address 10.3.0.3 255.255.255.0
 no shutdown
!
router ospf 1
 network 10.0.0.0 0.0.0.255 area 0
 network 10.0.4.0 0.0.0.255 area 0
 redistribute eigrp 100 subnets
!
router eigrp 100
 network 10.3.0.0 0.0.0.255
 network 192.168.3.0 0.0.0.255
 redistribute ospf 1 metric 10000 100 255 1 1500
 interface GigabitEthernet0/2
 ip summary-address eigrp 100 0.0.0.0 0.0.0.0
!
end
```

### 3.3. Router2 (ABR: OSPF Area 0 $\leftrightarrow$ Area 1)

```cisco
hostname Router2
enable secret class
no ip domain-lookup
ip routing
!
interface GigabitEthernet0/0
 ip address 10.0.3.2 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
interface GigabitEthernet0/1
 ip address 10.0.4.2 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
interface GigabitEthernet0/2
 ip address 10.2.0.2 255.255.255.0
 ip ospf 1 area 1
 no shutdown
!
router ospf 1
 network 10.0.3.0 0.0.0.255 area 0
 network 10.0.4.0 0.0.0.255 area 0
 area 1 stub
!
end
```

### 3.4. Router8 (HSRP Active - VLAN 80)

```cisco
hostname Router8
enable secret class
no ip domain-lookup
ip routing
!
interface GigabitEthernet0/1
 ip address 10.2.0.8 255.255.255.0
 ip ospf 1 area 1
 no shutdown
!
interface GigabitEthernet0/0
 no ip address
 no shutdown
!
interface GigabitEthernet0/0.80
 encapsulation dot1Q 80
 ip address 192.168.8.2 255.255.255.0
 standby 80 ip 192.168.8.1
 standby 80 priority 150
 standby 80 preempt
!
interface GigabitEthernet0/0.81
 encapsulation dot1Q 81
 ip address 192.168.81.2 255.255.255.0
 standby 81 ip 192.168.81.1
 standby 81 priority 100
 standby 81 preempt
!
router ospf 1
 area 1 stub
 network 10.2.0.0 0.0.0.255 area 1
 network 192.168.8.0 0.0.0.255 area 1
 network 192.168.81.0 0.0.0.255 area 1
!
end
```

### 3.5. Switch12 (HSRP Distribution Trunk)

```cisco
hostname Switch12
enable secret class
!
vlan 80
 name Staff
vlan 81
 name Guest
!
interface GigabitEthernet0/1
 switchport mode trunk
!
interface GigabitEthernet0/2
 switchport mode trunk
!
interface FastEthernet0/1
 switchport mode trunk
!
interface FastEthernet0/2
 switchport mode trunk
!
end
```