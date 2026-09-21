# Giới thiệu Neutron
\- OpenStack Networking service cung cấp API cho phép users thiết lập và định nghĩa kết nối và địa chỉ trong cloud. Project chịu trách nhiện cho Networking services là **neutron**. OpenStack Networking xử lý tạo và quản lý cơ sở hạ tầng mạng ảo, bao gồm networks, switches, subnets, và router cho devices được quản lý bởi OpenStack Compute service (Nova). Các services nâng cao như firewalls hoặc virtual private networks (VPNs) có thể được sử dụng.  
\- OpenStack Networking gồm neutron-server, database cho lưu trữ persistent, và bất kỳ số plug-in agent, cung cấp 1 số serviceskhsc như giao tiếp với native Linux networking mechanisms, external devices và SDN controller.  
\- OpenStack Networking là hoàn toàn độc lập và có thể triển khai trên 1 server độc lập.  
\- OpenStack Networking tích với 1 số thành phần OpenStac khác:  
- OpenStack Identity service (keystone) được sử dụng cho việc xác thực và ủy quyề của API request.
- OpenStack Compute service (nova) được sử dụng cho việc cắm mõi virtual NIC trên VM đến 1 mạng cụ thể.
- OpenStack Dashboard (horizon) được sử dụng bởi admin và user để tạo và quản lý network service thông qua giao diện web.
  # 1.Cơ bản Networking

<a name="1.1"></a>

## 1.1.Ethernet
\- Ethernet là giao thức mạng, được định nghĩa vởi tiêu chuẩnIEEE 802.3. Hầu hết các network interface cards (NICs) truyề thông bằng Ethernet.  
\- Trong mô hình OSI của giao thức mạng, Ethernet thuộc layer 2 (data link layer).  
\- Trong mạng Ethernet, các hosts kết nối với mạng truyề thông bằng cách trao đổi các frames. Mỗi host trên mạng Ethernet có 1 địa chỉ MAC duy nhất. Cụ thể, mỗi virtual machine (VM) trong môi trường OpenStack có 1 địa chỉ MAC duy nhất.  
\- Ngày trước, mạng Ethernet như một bus đơn mà mỗi hosts kết nối với nhau. Tuy nhiên, ngày nay, người ta đã thay thế bằng switch, trong đó các hosts kến nối trực tiếp đến switch.  
\- Trong mạng Ethernet, mỗi host trên mạng có thể gửi frame trực tiếp đến host khác. Mạng Ethernet cũng hỗ trợ broadcast để 1 host có thể gửi frame đến mỗi host khác bằng cách chỉ định địa chỉ MAC đích là ff:ff:ff:ff:ff:ff. ARP và DHCP là 2 giao thức được sử dụng trong Ethernet broadcast. Vì Ethernet network hỗ trợ broadcast nên người ta còn gọi mạng Ethernet là **brodcast domain**.  
\- Khi NIC nhận được Ethernet frame, mặc định NIC kiểm tra địa chỉ MAC định có khớp với địa chỉ của NIC (hoặc địa chỉ broadcast), và Ethernet frame bị hủy nếu không phù hợp.  
Đối với compute host, hành vi này là không phù hợp bởi vì farme có thể được dành cho 1 trong các trường hợp sau:  
- Các Nics có thể được cấu hình ở `promiscuous mode`, nơi chúng truyền tất cả Ethernet frames đến hệ điều hành, ngay cả khi địa chỉ MAC không khớp. Compute hosts sẽ luôn phải có các NIC thích hợp được cấu hình cho `promiscuous mode`.  
## 1.2.VLAN
\- VLAN là công nghệ mạng cho phép 1 switch hoạt động như thể nó là nhiều switch độc lập. Cụ thể, 2 hosts kết nối đến cùng 1 switch nhưng trên VLANs khác nhau sẽ không thể giao tiếp với nhau. OpenStack tạn dụng VLANs để cô lập lưu lượng giữa các projects khác nhau, kể cả các projects trên cùng 1 compute host. Mỗi VLAN có 1 ID, từ 1 đến 4095.  
\- 1 switchport được cấu hình để truyề frames từ tất cả các VLAN IDs được gọi là **trunk port**. IEEE 802.1Q là tiêu chuẩn mạng mô tả cách thức VLAN tags được mã hóa trong các khung Ethernet khi trunking được sử dụng.  
\- Chú ý: Nếu bạn sử dụng VLANs trên physical switches để cô lập project trong OpenStack cloud, bạn phải đảm bảo rằng tất cả các switchports của bạn được cấu hình như các cổng trunks.  

<a name="1.3"></a>

## 1.3.Subnets và ARP
\- Trong khi NICs sử dụng địa chỉ MAC để xác định địa chỉ mạng hosts, ứng dụng TCP/IP sử dụng địa chỉ IP. Address Resolution Protocol (ARP) là cầu nối giữa Ethernet và IP bằng cách dịch địa chỉ IP thành địa chỉ MAC.  
\- Địa chỉ IP được chia thành 2 phần: `network number` và `host identifier`. 2 hosts nằm trên cùng 1 **subnet** nếu chúng có cùng 1 `network number`. 2 host chỉ có thể truyền thông trực tiếp qua Ethernet nếu chúng ở trên cùng 1 local network. ARP giả định tất cả PC trong cùng 1 subnet đều nằm trong 1 local network. Người qunar trị mạng phải gắn địa chỉ IP và netmasks cho hosts để 2 hosts nằm trong cùng 1 subnet đều nằm trên cùng 1 local network, nếu không ARP không hoạt động đúng.  
\- Có 2 cú phép để thể hiện netmask:  
- dotted quad (VD: 255.255.255.0)
- classless inter-domain routing (CIDR) (VD: 192.168.1.5/24)


<a name="1.4"></a>

## 1.4.IP
\- Internet Protocol (IP) xác định cách định tuyến giữa các hosts mà được kết nối với local networks khác nhau. IP dựa trên network host đặc biệt gọi là routers hoặc gateway. 1 router là 1 host mà được kết nối đến ít nhất 2 local network và có thể forward IP packets từ 1 local network này đến local network khác. 1 router có nhiều địa chỉ IP, 1 cho mỗi mạng nó được kết nối.  
\- Trong mô hình OSI của giao thức mạng, giao thức IP nằm ở layer 3 (network layer).  
\- 1 host gửi packet đến địa chỉ IP sẽ tham khảo **routing table** của nó để xác định máy tính nào trên local network sẽ được nhận packet này. **Routing table** duy trì danh sách các  subnets liên kết với mỗi local network mà host được kết nối trực tiếp, cũng như 1 danh sách của định tuyến nằm trên các local network.  
\- Trên hệ điều hành Linux, có 1 vài command để show bảng routing:  
```
ip route show
route -r
netstat -rn
```


<a name="1.5"></a>

## 1.5.TCP/UDP/ICMP
\- **Transmission Control Protocol** (TCP) là giao thức lớp 4 được sử dụng phổ biến trong các ứng dụng mạng. TCP là giao thức connection-oriented: nó sử dụng mô hình client-server. Sự tương tác dựa trên TCP tiến hành như sau:  
- 1.Client kết nối đến server.
- 2.Client và server trao đổi dữ liệu.
- 3.Client hoặc server hủy kết nối.

\- Bởi vì host có nhiều ứng dụng chạy trên TCP, TCP sử dụng **ports** để xác định duy nhất các ứng dụng dựa trên TCP. 1 TCP port là 1 số trong khoảng 1-65535, chỉ 1 ứng dụng trên host có thể liên kết với 1 TCP port tại 1 thời điểm.  
\- TCP server lắng nghe trên port. Ví dụ: SSH server lắng nghe trên port 22. Client kết nối đến server sử dụng TCP, client phải biết địa chỉ IP của server và TCP port của server.  
\- Hệ điều hành của ứng dụng TCP client sẽ tự động gán 1 port number cho client. Client sẽ sở hữu port number này cho đến khi kết nối TCP được chấm dứt, sau đố hệ điều hành phục gồi port number đó. Loại ports này gọi là **ephemeral port**.  
\- IANA duy trì việc đăng kí port number cho nhiều dịch vụ dựa trên TCP. Việc đăng kí Tcp port number là không bắt buộc, nhưng đăng kí port number sẽ tránh được va chạm với các dịch vụ khác. TCP ports mặc định được sử dụng cho các dịch vụ khác nhau tham giao vào việc triển khai OpenStack.  
\- API phổ biến nhất để viết các ứng dụng dựa trên TCP là **Berkeley sockets**, được gọi là BSD sockets hoặc sockets. TCP là giao thức đáng tin cậy vì nó sẽ truyền lại packets bị mất hoặc bị lỗi trên đường truyền.  
\- UDO là giao thức lớp 5 khác. UDP là giao thức **connectionless**: 2 ứng dụng giao tiếp qua UDP không cần thiết lập kết nối trước khi trao đổi dữ liệu. UDP là giao thức không đáng tin cậy. Hệ điều hành không phát lại hoặc thậm chí không phát hiện UDP packets bị mất. Hệ điều hành cũng không đảm bảo rằng ứng dụng nhận các UDP packet theo đúng thứ tự mà chúng được gửi.  
\- UDP và TCP đều sử dụng ports để phần biệt các ứng dụng khác nhau đang chạy trên cùng hệ thống. Tuy nhiên, hệ điều hành xử lý UDP port tách biệt với TCp port. Ví dụ: 1 ứng dụng được liên kết với TCP port 16543 và 1 ứng dụng được liên kết với UDP port 16543. 
\- Tương tự TCP, sockets API là API phổ biến để viết các ứng dụng dựa trên UDP.  
\- DHCP, DNS, Network Time Protocol (NTP), và VXLAN là những giao thức dựa trên UDP trong môi trường OpenStack.  
\- ICMP là giao thức được sử dụng để gửi các thông điệp điều khiển qua mạng IP.  
VD: 1 router nhận được gói tin IP có thể gửi gói tin ICMP về nguồn nếu không có tuyến đường trong bảng routing của router tương ứng với địa chỉ đích hoặc gói tin quá lớn cho router xử lý.  


<a name="2"></a>

# 2.Các thành phần mạng
<a name="2.1"></a>

## 2.1.Switches
Switches là thiết bị Multi-Input Multi-Output (MIMO) cho phép truyền packet từ 1 node đến 1 node khác. Switches hoạt động ở lớp 2 trong mô hình mạng. Chúng chuyển tiếp lưu lượng dựa trên địa chỉ Ethernet đích trong packet header.  

<a name="2.2"></a>

## 2.2.Router
Router là thiết bị cho phép packet đi từ mạng lớp 3 này đến mạng lớp 3 khác. Router cho phéo giao tiếp giữa 2 node trên các mạng lớp 3 khác nhau không kết nối trực tiếp với nhau. Router hoạt động ở lớp 3 trong mô hình 3.Chúng định tuyến dựa trên địa chỉ IP đích trong packet header.

<a name="2.3"></a>

## 2.3.Firewalls
Firewalls được sử dụng để điều khiển lưu lượng truy cập đến và đi từ máy chủ hoặc mạng. Firewall có thể là thiết bị chuyên dụng hoặc software-based filtering mechanism implemented được hiện bởi hệ điều hành. Firewall được sử dụng để hạn chế lưu lượng truy cập đến máy chủ dựa trên các rules được định nghĩa trên máy chủ. Chúng lọc packet dựa trên địa chỉ IP nguồn, địa chỉ IP đích, port number, trạng thái kết nối, etc. Cacs hệ điều hành Linux thực hiện tường lửa thông qua iptables.

<a name="2.4"></a>

## 2.4. Load balancers
Load balancers có thể là phần mềm hoặc thiết bị phần cứng cho phép lưu lượng truy cập được phần phối đồng đều trên nhiều server. Bằng cách phần phối lưu lượng truy cập trên nhiều server, nó tránh quá tải cho 1 server.  

<a name="3"></a>

# 3.Overlay (tunnel) protocols
<a name="3.1"></a>

## 3.1.Generic routing encapsulation (GRE)
Generic routing encapsulation (GRE) là giao thức chạy trên IP và được sử dụng để tạo kết nối point-to-point bảo mật. 

<a name="3.2"></a>

## 3.2.Virtual extensible local area network (VXLAN)
VXLAN là giao thức overlay lớp 2 qua mạng lớp 3.

<a name="4"></a>

# 4.Network namspaces
<a name="4.1"></a>

## 4.1.Linux network namespaces
\- Network namespaces là 1 trong 7 loại đã nói ở phần trên. Network namespaces ảo hóa mạng. Trên mỗi network namespaces chứa duy nhất 1 loopback interface.  
\- Mỗi network interface (physical hoặc virtual) có duy nhất 1 namespaces và có thể di chuyển giữa các namespaces.  
\- Mỗi namespaces có 1 bộ địa chỉ IP, bảng routing, danh sách socket, firewall và các nguồn tài nguyên mạng riêng.  
\- Khi network namespaces bị hủy, nó sẽ hủy tất cả các virtual interfaces nào bên trong nó và di chuyển bất kỳ physical interfaces nào trở lại network namespaces root.  

<a name="4.2"></a>

## 4.2.Virtual routing and forwarding (VRF)
Trong networking, khái niệm tương tự network namespaces của Linux là VRF - Virtual Routing and Forwarding, là một tính năng cấu hình được trên các router như của Cisco hoặc Alcatel-Lucent, Juniper,...VRF là một công nghệ IP cho phép tồn tại cùng một lúc nhiều routing instance trong cùng 1 router ở cùng một thời điểm (multiple instances of a routing table). Do các routing instances này là độc lập nên nó cho phép sự chồng lấn về địa chỉ IP subnet trên các intefaces của router mà không gặp tình trạng xung đột. Có thể hiểu VRF giống như VMWare cho router vậy, còn các routing instances tương tự như các VMware guest instances, hoặc cũng có thể hiểu nó tương tự như VLANs tuy nhiên VRF hoạt động ở layer 3.

<a name="5"></a>

# 5.Network address translation
\- NAT là quá trình thay đổi địa chỉ nguồn hoặc địa chỉ đích trong header của gói tin IP. Nói chung, các ứng dụng của người gửi và người nhận không nhận biết được rằng gói tIP đnag được thao tác.  
\- NAT thường được thực hiện bởi router. Tuy nhiên, trong OpenStack thường là Linux server thực hiện chức năng NAT, không phải hardware router. Server sử dụng gói phần mềm iptables để thực hiện chức năng NAT.  

<a name="5.1"></a>

## 5.1.SNAT
\- **Source Network Translation** (SNAT) , NAT router sửa đổi địa chỉ IP của bên gửi trong gói tin IP. SNAT thường được sử dụng để cho phép các địa chỉ private kết nối với servers trên public Internet.  
\- RFC 1918 quy định 3 dỉa subnets sau là địa chỉ private:  
- 10.0.0.0/8
- 172.16.0.0/12
- 192.168.0.0/16

\- OpenStack sử dụng SNAT để cho phép các ứng dụng chạy bên trong instances có thể kết nối public Internet.  

<a name="5.2"></a>

## 5.2.DNAT
\- **Destination Network Address Translation** (DNAT), NAT router thay dổi địa chỉ IP đích trong header gói tin IP.  
\- OpenStack sử dụng DNAT để định tuyến gói tin từ instances đến OpenStack metadata service. 

<a name="5.3"></a>

## 5.3.One-to-one NAT
Trong **one-to-one NAT**, NAT router duy trù mapping giữa địa chỉ IP private và địa chỉ IP public. OpenStack sử dụng **one-to-one NAT** để thực hiện **địa chỉ IP floating**.  

<a name="6"></a>

## 6. OpenStack Networking

**OpenStack Networking** cho phép tạo và quản lý các đối tượng mạng (**network objects**), tiêu biểu:

- **Networks**
- **Subnets**
- **Ports**
- **Routers**
- **Security groups**
- Các dịch vụ mạng mở rộng như **Load Balancing**, **Firewall** và **VPN**

Dịch vụ Networking của OpenStack có tên là **Neutron**.

### 6.0.1. Vai trò của Neutron

Neutron cung cấp API để xác định và quản lý:

- Kết nối mạng cho các tài nguyên trong cloud.
- Địa chỉ IP.
- Layer 2 networking.
- Layer 3 forwarding và routing.
- NAT.
- Load balancing.
- Firewall.
- Virtual Private Network.

Neutron cũng cho phép OpenStack tích hợp với nhiều công nghệ networking khác nhau thông qua **plug-in** và **agent**.

### 6.0.2. Các thành phần chính

#### API Server

OpenStack Networking API hỗ trợ:

- **Layer 2 networking**
- **IP Address Management (IPAM)**
- **Layer 3 routing**

Layer 3 router cho phép thực hiện routing giữa các Layer 2 networks.

Neutron có nhiều plug-in để tương tác với các công nghệ mạng khác nhau, chẳng hạn:

- Routers
- Virtual switches
- Software-Defined Networking (SDN) controllers

#### Plug-in và Agents

Plug-in và agent thực hiện các thao tác như:

- Plug/unplug port.
- Tạo network.
- Tạo subnet.
- Cấp phát địa chỉ IP.
- Thực hiện các thao tác mạng tại compute/network node.

Việc lựa chọn plug-in và agent phụ thuộc vào nhà cung cấp và công nghệ mạng được sử dụng trong cloud.

> **Lưu ý:** Theo tài liệu gốc, tại một thời điểm chỉ sử dụng một plug-in.

#### Messaging Queue

Message queue đảm nhiệm việc nhận và định tuyến các yêu cầu **RPC (Remote Procedure Call)** giữa các agent để hoàn thành các thao tác API.

Trong kiến trúc **ML2 (Modular Layer 2)**, message queue được sử dụng cho RPC giữa:

- `neutron-server`
- Các `neutron-agent` chạy trên hypervisor

ML2 cũng sử dụng **mechanism drivers**, ví dụ:

- Open vSwitch
- Linux Bridge

---

# 6.1. Khái niệm

Để xây dựng topology mạng phong phú, người quản trị hoặc người dùng có thể tạo và cấu hình:

- Networks
- Subnets
- Ports
- Routers

Sau đó, các dịch vụ OpenStack khác, đặc biệt là **OpenStack Compute (Nova)**, có thể gắn **virtual devices** của instance vào các port thuộc những network này.

### Quan hệ cơ bản

```text
OpenStack Compute (Nova)
          |
          | gắn virtual interface
          v
        Port
          |
          v
       Network
          |
          v
       Subnet
```

**OpenStack Compute** sử dụng **OpenStack Networking** để cung cấp kết nối mạng cho instance.

Neutron cho phép mỗi project có nhiều private network. Các project có thể lựa chọn không gian địa chỉ IP riêng, kể cả trong trường hợp các dải IP của những project khác bị trùng nhau.

Có hai loại network chính được đề cập trong tài liệu:

1. **Provider Network**
2. **Self-Service / Project Network**

---

# 6.1.1. Provider Networks

**Provider network** cung cấp khả năng kết nối **Layer 2** trực tiếp đến instance, với tùy chọn hỗ trợ:

- DHCP
- Metadata service

Provider network được map với các mạng Layer 2 đã tồn tại trong data center.

Một công nghệ thường được sử dụng là **VLAN tagging (802.1Q)** để xác định và phân tách các mạng.

## Đặc điểm

Provider network thường mang lại:

- Kiến trúc đơn giản hơn.
- Hiệu năng tốt.
- Độ tin cậy cao.
- Ít lớp virtual networking hơn.

Đổi lại, provider network có ít tính linh hoạt hơn vì nó phụ thuộc vào cấu hình của hạ tầng mạng vật lý.

Theo mặc định, chỉ **admin** mới có thể tạo hoặc cập nhật provider network vì việc cấu hình provider network có thể liên quan trực tiếp đến hạ tầng mạng vật lý.

Trong tài liệu gốc, các quyền policy liên quan gồm:

```text
create_network:provider:physical_network
update_network:provider:physical_network
```

> Việc cho phép tạo hoặc sửa provider network có thể cho phép sử dụng các tài nguyên mạng vật lý như VLAN. Vì vậy, quyền này nên được cấp cho các tenant/project đáng tin cậy.

## Hạn chế

Provider network chủ yếu xử lý kết nối **Layer 2** cho instance.

Theo tài liệu gốc, provider network không trực tiếp cung cấp các chức năng như:

- Virtual router
- Floating IP

## Vì sao provider network có thể có hiệu năng tốt?

Provider network có thể đưa các hoạt động Layer 3 ra hạ tầng mạng vật lý thay vì xử lý toàn bộ qua các thành phần Layer 3 virtual của OpenStack.

Điều này có thể giúp giảm tải cho các thành phần mạng của OpenStack và tận dụng khả năng routing của thiết bị mạng vật lý.

## Trường hợp sử dụng

Provider network phù hợp với những môi trường mà:

- OpenStack coexist với bare-metal hosts.
- Có sẵn một hạ tầng mạng vật lý lớn.
- Instance cần truy cập trực tiếp Layer 2.
- Ứng dụng cần sử dụng VLAN để kết nối với hệ thống bên ngoài OpenStack.

---

# 6.1.2. Routed Provider Network

**Routed provider network** cung cấp kết nối **Layer 3** tới instance.

Network này được map với các mạng Layer 3 đã tồn tại trong data center.

Về mặt cấu trúc, một routed provider network có thể map tới nhiều **Layer 2 segments**.

Có thể hình dung:

```text
Routed Provider Network
        |
        +---- L2 Segment 1
        |
        +---- L2 Segment 2
        |
        +---- L2 Segment 3
```

Mỗi Layer 2 segment về cơ bản tương ứng với một provider network segment.

---

# 6.1.3. Self-Service Networks

**Self-service network** chủ yếu cho phép các project thông thường (không có đặc quyền admin) tự quản lý mạng mà không cần admin trực tiếp tạo network cho họ.

Các mạng này chủ yếu là **virtual network** và thường cần **virtual router** để giao tiếp với:

- Provider network
- Mạng bên ngoài
- Internet

Self-service network cũng thường cung cấp:

- DHCP
- Metadata service

cho các instance.

## Overlay network

Trong nhiều triển khai, self-service network sử dụng các **overlay protocol** như:

- VXLAN
- GRE

Các giao thức này cho phép tạo nhiều mạng logic hơn so với việc chỉ dựa trên VLAN tagging `802.1Q`.

Một ưu điểm quan trọng của overlay là có thể tạo topology mạng logic độc lập tương đối với topology Layer 2 vật lý bên dưới.

Trong khi đó, VLAN thường yêu cầu cấu hình bổ sung trên hạ tầng mạng vật lý.

---

## IPv4 Self-Service Network

IPv4 self-service network thường sử dụng **private IP address**.

Khi giao tiếp ra provider/external network, traffic có thể đi qua **Source NAT (SNAT)** trên virtual router.

```text
Instance
   |
Private IP
   |
Self-Service Network
   |
Virtual Router
   |
SNAT
   |
Provider / External Network
```

### Floating IP

**Floating IP** cho phép truy cập instance từ provider/external network.

Traffic đi vào có thể sử dụng **Destination NAT (DNAT)** trên virtual router để chuyển tiếp tới private IP của instance.

```text
External Network
      |
 Floating IP
      |
 Virtual Router
      |
    DNAT
      |
Private IP
      |
   Instance
```

---

## IPv6 Self-Service Network

Theo tài liệu gốc, IPv6 self-service network sử dụng dải địa chỉ IP public và tương tác với provider network thông qua **virtual router** với **static route**.

---

## Layer 3 Agent

Neutron thực hiện virtual router thông qua **Layer 3 agent**.

Trong một triển khai thông thường, Layer 3 agent chạy trên ít nhất một **network node**.

Điểm khác biệt quan trọng:

| Provider Network | Self-Service Network |
|---|---|
| Chủ yếu cung cấp kết nối L2 | Có virtual networking đầy đủ hơn |
| Kết nối trực tiếp với physical network | Thường sử dụng virtual router |
| Có thể tận dụng routing vật lý | Thường cần L3 agent |
| Ít phụ thuộc vào virtual L3 | Traffic giữa các mạng cần routing |

---

## Project Network và Isolation

User có thể tạo project network để phục vụ kết nối bên trong project.

Theo mặc định:

- Project network được cô lập.
- Các project khác không thể sử dụng network nếu không được chia sẻ.
- Có thể sử dụng các công nghệ network isolation/overlay khác nhau.

---

## Các loại network technology

### Flat

Tất cả instance nằm trên cùng một network.

Đặc điểm:

- Không sử dụng VLAN tagging.
- Không có VLAN segmentation.
- Instance có thể chia sẻ cùng network với host tùy topology.

```text
Host A ----+
           |
Instance 1-+---- Flat Network
           |
Instance 2-+
```

### VLAN

Cho phép tạo nhiều provider hoặc project network dựa trên **VLAN ID (802.1Q)**.

VLAN ID trong OpenStack tương ứng với VLAN được cấu hình trên mạng vật lý.

```text
Physical Network
      |
      +--- VLAN 100 ---> Network A
      |
      +--- VLAN 200 ---> Network B
      |
      +--- VLAN 300 ---> Network C
```

### GRE và VXLAN

**GRE** và **VXLAN** là các giao thức encapsulation được sử dụng để tạo **overlay network**.

Overlay network cho phép các compute instance giao tiếp với nhau thông qua một mạng logic nằm phía trên network vật lý.

Khi cần:

- Giao tiếp giữa các project network.
- Đi ra ngoài overlay network.
- Kết nối project network với external network.

thì cần có router.

Router cũng có thể cung cấp khả năng truy cập instance từ external network thông qua **Floating IP**.

---

# 6.1.4. Subnets

**Subnet** là một khối địa chỉ IP cùng với các trạng thái/cấu hình mạng liên quan.

Subnet là một phần của cơ chế **IP Address Management (IPAM)** được Networking service cung cấp cho project network và provider network.

Subnet được sử dụng để phân bổ địa chỉ IP khi các **port** mới được tạo trên network.

Ví dụ:

```text
Network: private-net

Subnet:
192.168.10.0/24

Các IP có thể cấp phát:
192.168.10.1
192.168.10.2
192.168.10.3
...
192.168.10.254
```

Có thể hiểu đơn giản:

> **Network** là không gian mạng logic, còn **Subnet** xác định phạm vi địa chỉ IP nằm trong network đó.

---

# 6.1.5. Subnet Pools

End user có thể tạo subnet bằng các địa chỉ IP hợp lệ.

Tuy nhiên, trong một số môi trường, admin hoặc project muốn quy định trước một **pool địa chỉ IP** để subnet được tạo tự động từ pool đó.

**Subnet pool** cho phép:

- Giới hạn những dải địa chỉ IP được phép sử dụng.
- Đảm bảo subnet được tạo nằm trong pool đã định nghĩa.
- Hạn chế việc sử dụng trùng địa chỉ.
- Tránh việc hai subnet sử dụng chồng lấn địa chỉ từ cùng một pool.

Có thể hình dung:

```text
Subnet Pool
10.10.0.0/16
      |
      +---- Subnet A: 10.10.1.0/24
      |
      +---- Subnet B: 10.10.2.0/24
      |
      +---- Subnet C: 10.10.3.0/24
```

Tham khảo:

https://docs.openstack.org/ocata/networking-guide/config-subnet-pools.html#config-subnet-pools

---

# 6.1.6. Ports

**Port** là điểm kết nối dùng để gắn một thiết bị mạng, chẳng hạn NIC của virtual server, vào một virtual network.

Port cũng lưu trữ các thông tin cấu hình mạng liên quan, ví dụ:

- MAC address
- IP address
- Security group
- Trạng thái của port
- Các thuộc tính mạng liên quan

Có thể hình dung:

```text
VM
 |
Virtual NIC
 |
Port
 |
Network
 |
Subnet
```

Một cách nhớ đơn giản:

> **Port là “cổng kết nối” giữa thiết bị mạng của instance và OpenStack Network.**

---

# 6.1.7. Routers

**Router** cung cấp các dịch vụ virtual **Layer 3**, bao gồm:

- Routing
- NAT

Router có thể định tuyến giữa:

- Self-service network và provider network.
- Các self-service network với nhau, tùy cấu hình project.

Networking service sử dụng **Layer 3 agent** để quản lý router thông qua **network namespaces**.

Luồng khái quát:

```text
Private Network
      |
      v
Virtual Router
      |
      +---- Provider Network
      |
      +---- External Network
```

---

# 6.1.8. Security Groups

**Security group** cung cấp một tập hợp các **virtual firewall rules** để kiểm soát traffic tại mức port.

Security group kiểm soát hai hướng traffic:

- **Ingress** — traffic đi vào instance.
- **Egress** — traffic đi ra khỏi instance.

## Mô hình hoạt động

Security group sử dụng cơ chế **default deny**:

> Traffic không được rule cho phép sẽ bị từ chối.

Do đó, security group chủ yếu được xây dựng bằng các rule **allow** cụ thể.

Firewall driver sẽ chuyển các security group rules thành cấu hình cho cơ chế packet filtering bên dưới, ví dụ:

```text
Security Group Rule
        |
        v
Firewall Driver
        |
        v
Packet Filtering
(iptables hoặc cơ chế tương ứng)
```

---

## Default Security Group

Mỗi project có một security group mặc định tên là:

```text
default
```

Theo tài liệu gốc, security group này:

- Cho phép traffic egress mặc định.
- Từ chối traffic ingress mặc định.

Các rule của `default` security group có thể được thay đổi.

Nếu launch instance mà không chỉ định security group, `default` security group sẽ được tự động áp dụng.

Tương tự, nếu tạo port mà không chỉ định security group, `default` security group sẽ được áp dụng cho port đó.

---

## Metadata và Security Group

Nếu sử dụng **metadata service**, cần lưu ý rule egress tới:

```text
169.254.169.254:80
```

vì instance cần truy cập metadata service thông qua địa chỉ này.

Theo tài liệu gốc, nếu xóa các default egress rules và chặn TCP port 80 tới `169.254.169.254`, instance có thể không truy xuất được metadata.

---

## Stateful Security Group

Security group rules là **stateful**.

Ví dụ:

Nếu cho phép:

```text
Ingress TCP/22
```

cho SSH, traffic phản hồi của kết nối đã được thiết lập sẽ được xử lý theo trạng thái của connection mà không cần tạo một rule egress đối xứng riêng cho cùng phiên kết nối.

---

# 6.1.9. Extensions

Neutron có thể mở rộng chức năng thông qua các **extensions**.

Extensions cho phép Networking service cung cấp thêm các khả năng ngoài tập API/networking cơ bản.

Tham khảo:

https://docs.openstack.org/ocata/networking-guide/intro-os-networking.html

---

# 6.1.10. DHCP

**DHCP service** quản lý việc cấp phát địa chỉ IP cho instance trên:

- Provider network
- Self-service network

Networking service triển khai DHCP bằng **DHCP agent**.

DHCP agent quản lý các:

```text
qdhcp namespaces
```

và sử dụng:

```text
dnsmasq
```

để cung cấp DHCP service.

Có thể hình dung:

```text
Network
   |
   +---- qdhcp namespace
              |
              +---- dnsmasq
                       |
                       v
                  DHCP service
                       |
                       v
                    Instance
```

---

# 6.1.11. Metadata

**Metadata service** là một service tùy chọn cung cấp API để instance lấy metadata.

Ví dụ metadata có thể chứa:

- SSH keys
- Thông tin cấu hình instance
- Các thông tin được cung cấp cho instance trong quá trình khởi tạo

Instance có thể truy cập metadata service thông qua địa chỉ metadata service, thường được nhắc tới trong tài liệu là:

```text
169.254.169.254
```

---

# 6.2. Service và Component Hierarchy

Kiến trúc Networking có thể được nhìn theo các tầng:

```text
Neutron
│
├── Server
│   └── API / Database / Logic
│
├── Plug-ins
│   └── Quản lý / điều phối agents
│
├── Agents
│   ├── Layer 2
│   ├── Layer 3
│   ├── DHCP
│   └── Metadata
│
└── Services
    ├── Routing
    ├── VPNaaS
    ├── LBaaS
    └── FWaaS
```

---

# 6.2.1. Server

Neutron server chịu trách nhiệm cung cấp API và quản lý các thành phần backend, trong đó có database.

Có thể hiểu đơn giản:

> **Neutron Server là đầu mối tiếp nhận yêu cầu networking từ API và điều phối việc thực hiện yêu cầu đó.**

---

# 6.2.2. Plug-ins

Plug-in chịu trách nhiệm quản lý và phối hợp với các agent.

Trong kiến trúc **ML2**, việc lựa chọn cơ chế thực thi network được tổ chức thông qua các thành phần như:

- ML2 core plugin
- Type drivers
- Mechanism drivers

Tài liệu nguồn nhấn mạnh vai trò của plug-in trong việc tương tác với công nghệ networking bên dưới.

---

# 6.2.3. Agents

Agents chịu trách nhiệm thực hiện các tác vụ networking trên các node.

Các nhiệm vụ bao gồm:

- Cung cấp Layer 2/Layer 3 connectivity cho instance.
- Xử lý chuyển tiếp giữa physical network và virtual network.
- Xử lý DHCP.
- Xử lý metadata.
- Các tác vụ networking khác tùy loại agent.

---

## 6.2.3.1. Layer 2 — Ethernet và Switching

Các công nghệ được đề cập:

- **Linux Bridge**
- **Open vSwitch (OVS)**

Layer 2 chịu trách nhiệm chính về:

- Ethernet switching.
- Kết nối các interface/port.
- Chuyển tiếp frame trong mạng.

---

## 6.2.3.2. Layer 3 — IP và Routing

Các chức năng được đề cập:

- **L3 agent**
- **DHCP**

Layer 3 tập trung vào:

- IP routing.
- NAT.
- Kết nối giữa các network/subnet.
- Một phần cơ chế cấp phát IP thông qua DHCP.

---

## 6.2.3.3. Miscellaneous

Các chức năng bổ sung bao gồm:

- Metadata

---

# 6.2.4. Services

Networking service có thể cung cấp các dịch vụ mở rộng.

## 6.2.4.1. Routing Services

Routing services cung cấp khả năng định tuyến Layer 3 giữa các network.

Đây là thành phần quan trọng để self-service network có thể giao tiếp với provider/external network.

---

## 6.2.4.2. VPNaaS

**VPNaaS (Virtual Private Network-as-a-Service)** là phần mở rộng của Neutron nhằm cung cấp tính năng VPN.

Mục đích là cung cấp kết nối VPN thông qua OpenStack Networking thay vì phải cấu hình VPN thủ công cho từng môi trường.

---

## 6.2.4.3. LBaaS

**LBaaS (Load-Balancer-as-a-Service)** cung cấp API để tạo và cấu hình load balancer.

Tài liệu gốc đề cập implementation tham chiếu dựa trên:

**HAProxy**

Mô hình khái quát:

```text
Client
  |
  v
Load Balancer
  |
  +---- VM 1
  |
  +---- VM 2
  |
  +---- VM 3
```

Load balancer phân phối traffic tới các backend instance.

---

## 6.2.4.4. FWaaS

**FWaaS (Firewall-as-a-Service)** cung cấp API firewall để thử nghiệm và triển khai các chức năng firewall trong Networking service.

FWaaS được trình bày chi tiết ở phần tiếp theo.

---

# 7. Firewall-as-a-Service (FWaaS)

**Firewall-as-a-Service (FWaaS)** là chức năng firewall của OpenStack Networking.

Theo tài liệu gốc, FWaaS có thể áp dụng firewall cho các đối tượng OpenStack như:

- Projects
- Routers
- Router ports

> Tài liệu gốc được xây dựng theo bối cảnh OpenStack Ocata, vì vậy một số chi tiết về vị trí áp dụng firewall có thể khác với các phiên bản OpenStack hiện đại.

---

# 7.1. Các khái niệm cơ bản của Firewall

Hai khái niệm quan trọng là:

1. **Firewall Policy**
2. **Firewall Rule**

## Firewall Rule

Một firewall rule xác định các thuộc tính dùng để quyết định traffic có phù hợp với rule hay không.

Các thuộc tính có thể bao gồm:

- Protocol
- Port range
- Source IP
- Destination IP
- Action

Action có thể là:

- Allow
- Deny

Ví dụ khái quát:

```text
Protocol: TCP
Destination Port: 22
Source: 10.10.10.0/24
Action: Allow
```

Rule trên có thể được hiểu là cho phép SSH từ mạng `10.10.10.0/24`.

---

## Firewall Policy

**Policy** là tập hợp các firewall rules.

```text
Firewall Policy
│
├── Rule 1: Allow TCP/22
├── Rule 2: Allow TCP/80
├── Rule 3: Allow TCP/443
└── Rule 4: Deny other traffic
```

Policy có thể được public/share để nhiều project sử dụng, tùy cấu hình.

---

# 7.2. FWaaS Implementation

Cách firewall được thực thi phụ thuộc vào **driver**.

Ví dụ:

### iptables driver

Firewall được triển khai bằng các rule của:

```text
iptables
```

### Open vSwitch driver

Firewall rules có thể được thực thi thông qua các **flow entries** trong flow tables.

### Cisco firewall driver

Driver thao tác với các thiết bị firewall của Cisco.

### VMware driver

Driver có thể cấu hình:

```text
NSX router
```

---

# 7.3. FWaaS v1

FWaaS ban đầu được triển khai dưới dạng **FWaaS v1**.

FWaaS v1 cung cấp cơ chế bảo vệ ở **router level**.

Khi firewall được áp dụng cho một router, các internal port đi qua router đó sẽ nằm trong phạm vi bảo vệ của firewall.

Mô hình khái quát:

```text
                External Network
                       |
                       v
                +-------------+
                |   Router    |
                |   + FWaaS   |
                +-------------+
                 /           \
                /             \
              VM1             VM2
```

Tài liệu gốc có sơ đồ minh họa traffic ingress và egress cho VM2.

---

# 7.4. FWaaS v2

FWaaS v2 là triển khai mới hơn với khả năng chi tiết hơn.

Một thay đổi quan trọng là khái niệm **firewall group** được sử dụng để quản lý firewall thay cho mô hình firewall đơn giản của v1.

Firewall group có thể chứa:

- **Ingress policy**
- **Egress policy**

Mô hình:

```text
Firewall Group
│
├── Ingress Policy
│   ├── Rule 1
│   ├── Rule 2
│   └── ...
│
└── Egress Policy
    ├── Rule 1
    ├── Rule 2
    └── ...
```

Khác với FWaaS v1, FWaaS v2 cho phép firewall được áp dụng ở **port level** thay vì chỉ áp dụng ở router level.

Theo tài liệu gốc, router port có thể được chỉ định; đối với OpenStack Ocata, VM ports cũng được đề cập là có thể chỉ định.

---

# 8. Tổng kết kiến thức cần nhớ

## 8.1. Network → Subnet → Port → Instance

Đây là chuỗi rất quan trọng khi học Neutron:

```text
Network
   |
   +---- Subnet
   |
   +---- Port
            |
            +---- Virtual NIC
                       |
                       v
                     VM
```

### Network

Xác định **mạng logic**.

### Subnet

Xác định **dải IP** bên trong network.

### Port

Là **điểm kết nối** của thiết bị/virtual NIC vào network.

### Instance

VM sử dụng virtual NIC và port để tham gia vào network.

---

## 8.2. Provider Network vs Self-Service Network

```text
                    OpenStack
                        |
          +-------------+-------------+
          |                           |
          v                           v
 Provider Network              Self-Service Network
          |                           |
      Physical L2                 Virtual Network
          |                           |
       VLAN/L2                  VXLAN / GRE
          |                           |
          |                      Virtual Router
          |                           |
          +-------------+-------------+
                        |
                 External Network
```

### Provider Network

- Gần với physical network.
- Chủ yếu cung cấp Layer 2.
- Có thể sử dụng VLAN.
- Đơn giản và hiệu năng tốt.
- Phụ thuộc vào cấu hình physical network.

### Self-Service Network

- Do project/user quản lý.
- Là virtual network.
- Có thể sử dụng VXLAN/GRE.
- Thường sử dụng virtual router.
- Hỗ trợ NAT và Floating IP.
- Cung cấp isolation giữa các project.

---

## 8.3. Security Group vs FWaaS

Hai khái niệm dễ bị nhầm:

| Thành phần | Vai trò |
|---|---|
| **Security Group** | Kiểm soát traffic tại mức port/instance bằng security rules |
| **FWaaS** | Cung cấp firewall service ở mức mạng/router/port tùy phiên bản và triển khai |

Có thể nhớ:

```text
Security Group
      ↓
Port / Instance security

FWaaS
      ↓
Network firewall service
```

---

## 8.4. DHCP và Metadata

```text
                    Network
                       |
          +------------+------------+
          |                         |
          v                         v
        DHCP                    Metadata
          |                         |
      dnsmasq                169.254.169.254
          |                         |
          v                         v
       Instance                 Instance
```

### DHCP

Cấp phát IP và các thông tin mạng cho instance.

### Metadata

Cung cấp thông tin cấu hình/metadata cho instance, trong đó có thể bao gồm SSH key.

---

## 8.5. Các thành phần Neutron cần nhớ

| Thành phần | Chức năng chính |
|---|---|
| **Neutron Server** | API và điều phối Networking service |
| **ML2 / Plug-in** | Cơ chế tích hợp networking |
| **Mechanism Driver** | Thực thi network thông qua công nghệ cụ thể |
| **L2 Agent** | Switching / Layer 2 |
| **L3 Agent** | Routing / NAT |
| **DHCP Agent** | DHCP cho network |
| **Metadata Agent** | Cung cấp metadata cho instance |
| **OVS / Linux Bridge** | Công nghệ switching |
| **Router** | Kết nối và routing giữa các network |
| **Security Group** | Kiểm soát traffic ở mức port |
| **FWaaS** | Firewall service |
| **VPNaaS** | VPN service |
| **LBaaS** | Load balancing service |

---

# 9. Luồng khi troubleshooting Networking

Khi một VM không có mạng, có thể kiểm tra theo chuỗi:

```text
Instance
   ↓
Virtual NIC
   ↓
Port
   ↓
Security Group
   ↓
Subnet
   ↓
Network
   ↓
L2 Agent / OVS / Linux Bridge
   ↓
L3 Router
   ↓
DHCP / NAT
   ↓
Provider Network
   ↓
Physical Network
   ↓
External Network
```

Có thể chia troubleshooting thành các lớp:

### Layer 1 — Instance

Kiểm tra:

- VM có đang chạy không?
- VM có virtual NIC không?
- NIC có UP không?

### Layer 2 — Port / Switching

Kiểm tra:

- Port có tồn tại không?
- Port có thuộc đúng network không?
- MAC address có đúng không?
- OVS/Linux Bridge có nhận port không?

### Layer 3 — IP / Routing

Kiểm tra:

- Instance có IP không?
- Subnet có DHCP không?
- Router có kết nối đúng subnet không?
- Route có tồn tại không?
- NAT có đúng không?

### Security

Kiểm tra:

- Security group.
- Ingress rule.
- Egress rule.
- FWaaS nếu được sử dụng.

### External Connectivity

Kiểm tra:

- Provider network.
- VLAN.
- Physical switch.
- Router vật lý.
- External network.
- Floating IP nếu truy cập từ bên ngoài.

---

# 10. Cheat Sheet — Neutron

```text
NEUTRON
│
├── NETWORK
│   └── Mạng logic
│
├── SUBNET
│   └── Dải IP
│
├── PORT
│   └── Điểm kết nối của NIC
│
├── ROUTER
│   └── L3 / Routing / NAT
│
├── SECURITY GROUP
│   └── Firewall rules ở mức port
│
├── DHCP
│   └── Cấp IP
│
├── METADATA
│   └── Thông tin cấu hình cho VM
│
├── PROVIDER NETWORK
│   └── Kết nối với physical network
│
├── SELF-SERVICE NETWORK
│   └── Virtual / project network
│
├── L2 AGENT
│   └── Switching
│
├── L3 AGENT
│   └── Routing / NAT
│
└── EXTENSIONS
    ├── VPNaaS
    ├── LBaaS
    └── FWaaS
```

## Một câu để nhớ toàn bộ

> **Neutron quản lý network; subnet cung cấp dải IP; port kết nối NIC của VM vào network; L2 xử lý switching; L3 xử lý routing/NAT; DHCP cấp IP; Metadata cung cấp thông tin cho VM; Security Group kiểm soát traffic ở port; Provider Network nối OpenStack với physical network; Self-Service Network tạo mạng ảo cho project.**
