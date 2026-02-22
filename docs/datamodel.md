আগের চ্যাপ্টারে আমরা Nautobot 3.0 ইনস্টল করলাম। Nirvor Communication এর NOC টিম এখন একটা চালু Nautobot instance পেয়েছে। কিন্তু এখনও কোনো ডেটা নেই - খালি একটা সিস্টেম। ডেটা এন্ট্রি শুরু করার আগে বুঝতে হবে Nautobot কীভাবে নেটওয়ার্ক ডেটা স্টোর করে, কীভাবে বিভিন্ন জিনিস একে অপরের সাথে সম্পর্কিত।

এই চ্যাপ্টারে আমরা Nautobot 3.0 এর ডেটা মডেল বুঝব। এটা একটু থিওরি মনে হতে পারে, কিন্তু এই ফাউন্ডেশন না বুঝলে পরে ডেটা এন্ট্রি করতে গিয়ে সমস্যা হবে। Nirvor Communication এর নেটওয়ার্ককে উদাহরণ হিসেবে ব্যবহার করে প্রতিটা কনসেপ্ট ব্যাখ্যা করব।

### Nautobot 3.0 এর নতুন আর্কিটেকচার

Nautobot 3.0 এ সবচেয়ে বড় পরিবর্তন হলো এর modular architecture। পুরো সিস্টেমকে কয়েকটা Core Apps-এ ভাগ করা হয়েছে:

**কোর অ্যাপ্লিকেশন:**

- **Organization**: Tenants, Teams, Locations
- **DCIM** (Data Center Infrastructure Management): Devices, Racks, Cables
- **IPAM** (IP Address Management): IP Addresses, Prefixes, VLANs
- **Circuits**: Providers, Circuit Types, Circuits
- **Extras**: Roles, Status, Tags, Custom Fields

প্রতিটা App নিজের মতো কাজ করে, কিন্তু সবগুলো একসাথে মিলে একটা সম্পূর্ণ Network Source of Truth তৈরি করে।

চলুন একটা একটা করে দেখি।

---

### Organization App - নেটওয়ার্কের কাঠামো

Organization App হলো Nautobot এর ভিত্তি। এখানে আপনার নেটওয়ার্কের organizational structure define করবেন।

**Key Objects:**

- **Locations**
- **Location Types**  
- **Tenants**  
- **Tenant Groups** 
- **Teams**

---

#### Locations - সবচেয়ে গুরুত্বপূর্ণ পরিবর্তন

**Nautobot 2.x এ ছিল:** Sites (একটা ফ্ল্যাট লিস্ট)

**Nautobot 3.0 এ এখন:** Locations (hierarchical, nestable)

Nirvor Communication এর নেটওয়ার্ক ঢাকা নর্থ জোনে ছড়িয়ে আছে। তাদের এই স্ট্রাকচার:

```
Nirvor Communication Network
  └── Dhaka North Zone
       ├── Mirpur Cluster
       │    └── Mirpur POP
       │         ├── Rack A
       │         └── Rack B
       └── Uttara Cluster
            └── Uttara POP
                 └── Rack A
```

এই hierarchy Nautobot 3.0 এ সহজেই represent করা যায়।

**Location Types:**

প্রথমে define করতে হয় কী ধরনের locations আছে:

```
Location Type: Zone
  - Description: Geographic coverage zone
  - Nestable: Yes
  - Parent: None

Location Type: Cluster
  - Description: Group of nearby POPs
  - Nestable: Yes
  - Parent: Zone

Location Type: POP
  - Description: Point of Presence
  - Nestable: Yes
  - Parent: Cluster

```

**Actual Locations:**

এরপর actual locations তৈরি করবেন:

```
Location: Dhaka North Zone
  - Type: Zone
  - Parent: None
  - Status: Active

Location: Mirpur Cluster
  - Type: Cluster
  - Parent: Dhaka North Zone
  - Status: Active

Location: Mirpur POP
  - Type: POP
  - Parent: Mirpur Cluster
  - Status: Active
  - Address: House 25, Road 10, Mirpur-12, Dhaka-1216
  - GPS: 23.8103, 90.3654

Rack (DCIM Object):

Rack: Rack A
  - Location: Mirpur POP
  - Status: Active
  - Height: 42U
```

**Location Hierarchy এর সুবিধা:**

১. **Filtering সহজ:** "Mirpur Cluster এর সব ডিভাইস দেখাও" - এক ক্লিকে
২. **Reporting:** Zone অনুযায়ী রিপোর্ট তৈরি করা সহজ
৩. **Scalability:** নতুন zone/cluster যোগ করা সহজ
৪. **Clarity:** নেটওয়ার্ক স্ট্রাকচার একদম পরিষ্কার

#### Tenants - মাল্টি-টেন্যান্সি

যদি আপনার একাধিক ব্যবসা বা ব্র্যান্ড থাকে, তাহলে Tenants ব্যবহার করবেন।

Nirvor Communication এর দুটো ব্র্যান্ড আছে:
- Nirvor Communicatio (Residential)
- Nirvor Communication (Corporate)

```
Tenant: Nirvor Communication Residential
  - Description: Residential broadband services
  - Group: Consumer Services

Tenant: Nirvor Communication Business
  - Description: Corporate connectivity solutions
  - Group: Enterprise Services
```

প্রতিটা device, IP address, circuit-এ tenant assign করা যায়। তাহলে আলাদা আলাদা করে ট্র্যাক করতে পারবেন।

**ছোট ISP এর জন্য:** Tenants সাধারণত লাগে না। সব একটা single organization এর আন্ডারে রাখলেই চলে।

#### Teams - কে কী দেখবে

বড় organization এ বিভিন্ন টিম থাকে। Nautobot 3.0 এ Teams দিয়ে সেটা manage করা যায়।

Nirvor Communication এর টিম:

```
Team: NOC Team
  - Members: Jahangir, Asif, Rahim
  - Permissions: সব দেখতে এবং এডিট করতে পারবে

Team: Field Operations
  - Members: Karim, Salman, Rafiq
  - Permissions: শুধু devices এবং cables দেখতে/এডিট করতে পারবে

Team: Management
  - Members: CEO, CTO
  - Permissions: শুধু read-only access
```

---

### DCIM App - ফিজিক্যাল ইনফ্রাস্ট্রাকচার

DCIM (Data Center Infrastructure Management) হলো Nautobot এর সবচেয়ে বড় অংশ। এখানে আপনার সব physical network equipment থাকবে।

#### Device Types - ব্লুপ্রিন্ট

কোনো device যোগ করার আগে তার "type" define করতে হয়। Device Type হলো একটা blueprint বা template।

Nirvor Communication MikroTik router ব্যবহার করে। তাদের main model হলো CCR2004-1G-12S+2XS।

```
Device Type: MikroTik CCR2004-1G-12S+2XS
  - Manufacturer: MikroTik
  - Model: CCR2004-1G-12S+2XS
  - Part Number: CCR2004-1G-12S+2XS
  - Height: 1U
  - Full Depth: Yes
  - Subdevice Role: None (এটা parent device)
  
  Interface Templates:
    - ether1 (Type: 1000BASE-T)
    - sfp-sfpplus1 (Type: 10GBASE-X-SFP+)
    - sfp-sfpplus2 (Type: 10GBASE-X-SFP+)
    - ... sfp-sfpplus12 পর্যন্ত
    - sfp28-1 (Type: 25GBASE-X-SFP28)
    - sfp28-2 (Type: 25GBASE-X-SFP28)
```

**Interface Templates এর গুরুত্ব:**

যখন এই device type এর একটা নতুন device তৈরি করবেন, তখন সব interfaces automatically তৈরি হয়ে যাবে। ম্যানুয়ালি একটা একটা করে করতে হবে না।

#### Manufacturers

Device Type তৈরির আগে Manufacturer তৈরি করতে হয়:

```
Manufacturer: MikroTik
  - Description: Latvian network equipment manufacturer
  - Website: https://mikrotik.com

Manufacturer: TP-Link
  - Description: Chinese networking equipment
  - Website: https://www.tp-link.com

Manufacturer: Cisco
  - Description: American networking giant
  - Website: https://www.cisco.com
```

#### Devices - আসল হার্ডওয়্যার

এখন actual devices তৈরি করা যাবে।

Nirvor Communication এর Mirpur POP এ একটা core router আছে:

```
Device: R-DN-MIR-CORE-01
  - Device Type: MikroTik CCR2004-1G-12S+2XS
  - Role: Core Router (এটা Extras থেকে আসবে, পরে দেখব)
  - Location: Mirpur POP
  - Rack: Rack A
  - Position: 25U
  - Face: Front
  - Status: Active
  - Serial Number: ABC1234MIR001
  - Asset Tag: NIRVOR-RTR-001
  - Comments: Primary core router for Mirpur POP
```

**Device Naming Convention:**

Nirvor Communication এর convention:
```
Format: [Type]-[Zone]-[Site]-[Role]-[Number]

R = Router
DN = Dhaka North
MIR = Mirpur
CORE = Core Router
01 = First device

উদাহরণ:
R-DN-MIR-CORE-01 = Router, Dhaka North, Mirpur, Core, #1
SW-DN-UTT-DIST-01 = Switch, Dhaka North, Uttara, Distribution, #1
```

#### Interfaces - কানেকশন পয়েন্ট

Device তৈরি হলে তার interfaces automatically তৈরি হয়েছে (Device Type এর template থেকে)। কিন্তু এগুলোতে আরো details যোগ করতে হয়।

Mirpur core router এর uplink interface:

```
Interface: sfp-sfpplus1
  - Device: R-DN-MIR-CORE-01
  - Type: 10GBASE-X-SFP+
  - Enabled: Yes
  - MTU: 1500
  - Speed: 10 Gbps
  - Description: "Uplink to BTCL"
  - Mode: Access (Trunk নয়)
```

#### Cables - ফিজিক্যাল কানেকশন

দুটো device একসাথে কানেক্ট করতে Cable object তৈরি করতে হয়।

Mirpur core router থেকে distribution switch এ একটা fiber cable:

```
Cable: CORE-TO-DIST-01
  - Termination A:
    * Device: R-DN-MIR-CORE-01
    * Interface: sfp-sfpplus2
  
  - Termination Z:
    * Device: SW-DN-MIR-DIST-01
    * Interface: sfp1
  
  - Type: Single-Mode Fiber (SMF)
  - Status: Connected
  - Color: Blue
  - Length: 5 m
  - Label: CORE-TO-DIST-01
```

**Cable Trace ফিচার:**

Cable তৈরি করার পর Nautobot এর "Trace" ফিচার দিয়ে পুরো physical path follow করতে পারবেন। যেমন একটা customer এর connection কোন কোন device দিয়ে গেছে - সব দেখা যায়।

#### Racks - সংগঠিত রাখা

Rack হলো একটা physical enclosure যেখানে devices mount করা থাকে।

```
Rack: Rack A
  - Location: Mirpur POP
  - Status: Active
  - Type: 4-post cabinet
  - Width: 19 inches
  - Height: 42U
  - Description: Primary equipment rack
  - Outer Width: 600 mm
  - Outer Depth: 1000 mm
```

**Rack Elevation:**

Nautobot এ একটা visual rack elevation দেখা যায়। কোন U position এ কোন device আছে - graphically দেখায়।

```
42U ┌──────────────┐
    │              │  (Empty)
    │              │
25U ├──────────────┤
    │ R-DN-MIR-... │  MikroTik CCR2004 (1U)
24U ├──────────────┤
    │              │
20U ├──────────────┤
    │ SW-DN-MIR-...│  TP-Link Switch (1U)
19U ├──────────────┤
    ...
```

---

### IPAM App - IP Address Management

নেটওয়ার্কের logical দিক - IP addresses, subnets, VLANs।

#### IP Namespaces - Isolated IP Spaces

Nautobot 3.0 এ Namespace একটা নতুন ফিচার। এটা দিয়ে একই IP range বিভিন্ন context এ ব্যবহার করা যায়।

**Nirvor Communication এর ক্ষেত্রে:**

তাদের শুধু একটা namespace দরকার:

```
Namespace: Global
  - Description: Primary IP namespace for Nirvor Communication
```

**কখন একাধিক namespace লাগে?**

ধরুন আপনার দুটো সম্পূর্ণ আলাদা নেটওয়ার্ক আছে যেগুলো কখনোই interconnect হবে না। তখন উভয়েই 10.0.0.0/8 ব্যবহার করতে পারবেন আলাদা namespace এ রেখে।

#### Prefixes - IP রেঞ্জ

Prefix হলো একটা IP subnet।

Nirvor Communication এর কাছে BTCL থেকে পাওয়া একটা /22 public IP block আছে:

```
Prefix: 103.125.40.0/22
  - Type: Container (এর ভেতরে আরো ছোট prefixes আছে)
  - Status: Active
  - Namespace: Global
  - Description: Assigned public IP block from BTCL
  - Location: (খালি - কারণ এটা overall block)
```

**Hierarchical Prefixes:**

এই /22 block কে ভাগ করা হয়েছে:

```
Parent: 103.125.40.0/22 (Container)
  ├── 103.125.40.0/24 (Network)
  │   - Location: Mirpur POP
  │   - VLAN: RESIDENTIAL_MIR
  │   - Description: Residential customers - Mirpur
  │
  ├── 103.125.41.0/24 (Network)
  │   - Location: Uttara POP
  │   - VLAN: RESIDENTIAL_UTT
  │   - Description: Residential customers - Uttara
  │
  ├── 103.125.42.0/25 (Network)
  │   - VLAN: CORPORATE
  │   - Description: Corporate clients
  │
  └── 103.125.42.128/25 (Network)
      - Description: Infrastructure IPs
```

**Prefix Types:**

- **Container:** শুধু organize করার জন্য, এর ভেতরে child prefixes আছে
- **Network:** আসল usable network, এখান থেকে IP allocate হয়
- **Pool:** Dynamic allocation এর জন্য (যেমন DHCP pool)

#### IP Addresses - Individual IPs

Individual IP addresses assign করা হয় devices এর interfaces এ।

Management IP Space:

```
Prefix: 10.10.0.0/16
  - Type: Container
  - Status: Active
  - Namespace: Global
  - Description: Management IP space
```
Uplink & Infrastructure IPs:
```
Prefix: 103.125.42.128/25
  - Type: Container
  - Status: Active
  - Namespace: Global
  - Description: Uplink & Infrastructure IPs
```
Mirpur core router এর loopback IP:
```
IP Address: 10.10.1.1/32
  - Status: Active
  - Role: Loopback
  - Namespace: Global
  - Parent Prefix: 10.10.0.0/16
  - Assigned to:
    * Device: R-DN-MIR-CORE-01
    * Interface: lo0
  - DNS Name: r-mir-core-01.nirvor.bd
  - Description: Management loopback
```

Uplink IP:

```
IP Address: 103.125.42.130/30
  - Status: Active
  - Role: Network
  - Namespace: Global
  - Parent Prefix: 103.125.42.128/25
  - Assigned to:
    * Device: R-DN-MIR-CORE-01
    * Interface: sfp-sfpplus1
  - DNS Name: r-mir-uplink.nirvor.bd
  - Description: BTCL uplink connection
```
#### Summary

Namespace Nautobot 3.0 এর IPAM মডেলের একটি গুরুত্বপূর্ণ অংশ। এটি overlapping IP design, multi-tenant architecture এবং VRF-based network কে পরিষ্কারভাবে manage করতে সাহায্য করে।

#### VLANs - Layer 2 Segmentation

VLAN দিয়ে একই physical network কে logical ভাগে আলাদা করা যায়।

Nirvor Communication এর VLAN structure:

```
VLAN 10: MGMT_MIRPUR
  - Location: Mirpur POP
  - Status: Active
  - Description: Management network for Mirpur devices

VLAN 20: UPLINK_MIRPUR
  - Location: Mirpur POP
  - Status: Active
  - Description: Uplink to BTCL

VLAN 100: RESIDENTIAL_MIR
  - Location: Mirpur POP
  - Status: Active
  - Description: Residential customer traffic - Mirpur

VLAN 200: CORPORATE
  - Location: Global
  - Status: Active
  - Description: Corporate clients across all sites
```

**VLAN Groups:**

যদি একই VLAN ID বিভিন্ন সাইটে ভিন্ন purpose এ ব্যবহার করতে চান, তাহলে VLAN Groups ব্যবহার করুন।

---

### Circuits App - আপস্ট্রিম কানেক্টিভিটি

যেকোনো ISP এর নিজের uplink provider থাকে। Nirvor Communication BTCL থেকে ইন্টারনেট নেয়।

#### Providers

প্রথমে provider define করুন:

```
Provider: BTCL
  - ASN: 17494
  - Account Number: NIRVOR-BTCL-2023-MIR
  - Portal URL: https://isp.btcl.gov.bd
  - NOC Contact: +880 2-9555555
  - Admin Contact: noc@btcl.gov.bd
  - Comments: Primary upstream provider
```

Nirvor Communication এর একটা secondary provider ও আছে:

```
Provider: Summit Communications
  - ASN: 45928
  - Account Number: SKY-SUMMIT-2024
  - Portal URL: https://portal.summitcommunications.net
  - Comments: Secondary/backup provider
```

#### Circuit Types

Circuit এর ধরন define করুন:

```
Circuit Type: Fiber Optic
  - Description: Fiber optic connectivity

Circuit Type: Microwave
  - Description: Point-to-point microwave link

Circuit Type: IPLC
  - Description: International Private Leased Circuit
```

#### Circuits - Actual Connections

এখন actual circuit:

```
Circuit: BTCL-MIR-001
  - Provider: BTCL
  - Type: Fiber Optic
  - Status: Active
  - Install Date: 2023-03-15
  - Commit Rate: 5000 Mbps (5 Gbps)
  - Description: Primary 5Gbps uplink from Mirpur POP
  - Comments:
    * Contract Start: 2023-03-01
    * Contract Duration: 2 years
    * Monthly Cost: BDT 400,000
    * Renewal Date: 2025-03-01
```

**Circuit Terminations:**

প্রতিটা circuit এর দুই প্রান্ত থাকে - A এবং Z।

```
Termination A (Provider Side):
  - Side: A
  - Port Speed: 5 Gbps
  - Upstream Speed: 5000 Mbps
  - Cross Connect ID: BTCL-XC-MIR-12345

Termination Z (Nirvor Communication Side):
  - Side: Z
  - Location: Mirpur POP
  - Device: R-DN-MIR-CORE-01
  - Interface: sfp-sfpplus1
  - Port Speed: 5 Gbps
```

এভাবে Nautobot জানে ঠিক কোন interface এ uplink আছে।

---

### Extras App - Flexibility এবং Customization

Extras app হলো Nautobot এর "toolbox"। এখানে এমন জিনিস আছে যা সব apps এ ব্যবহার হয়।

#### Roles - Unified System (Nautobot 3.0 এর বড় পরিবর্তন)

**Nautobot 2.x এ ছিল:**
- Device Roles (আলাদা)
- Rack Roles (আলাদা)
- IPAM Roles (আলাদা)

**Nautobot 3.0 এ এখন:**
- একটা unified Roles system
- Content Types দিয়ে নির্ধারণ করবেন কোথায় ব্যবহার হবে

Nirvor Communication এর Device Roles:

```
Role: Core Router
  - Content Types: dcim.device
  - Color: f44336 (Red)
  - Weight: 1000 (যত বেশি, তত গুরুত্বপূর্ণ)
  - Description: Core routing equipment

Role: Distribution Switch
  - Content Types: dcim.device
  - Color: ff9800 (Orange)
  - Weight: 800
  - Description: Distribution layer switches

Role: Access Switch
  - Content Types: dcim.device
  - Color: 4caf50 (Green)
  - Weight: 600
  - Description: Access layer switches
```

**একই Role একাধিক জায়গায়:**

```
Role: Critical Infrastructure
  - Content Types: dcim.device, dcim.rack, circuits.circuit
  - Color: 000000 (Black)
  - Weight: 2000
  - Description: Mission-critical components
```

এই role এখন devices, racks, এবং circuits তিনটাতেই ব্যবহার করা যাবে।

#### Status - Lifecycle Management

প্রতিটা object এর একটা status থাকে। Nautobot 3.0 এ custom status তৈরি করা যায়।

**Default Statuses:**
- Active
- Planned
- Staged
- Failed
- Inventory
- Decommissioned
- Offline
- Retired

Nirvor Communication একটা custom status যোগ করেছে:

```
Status: Under Maintenance
  - Content Types: dcim.device
  - Color: ffeb3b (Yellow)
  - Description: Device temporarily offline for maintenance
```

#### Tags - Categorization

Tags দিয়ে যেকোনো object categorize করা যায়।

Nirvor Communication এর কিছু tags:

```
Tag: production
  - Color: 4caf50 (Green)
  - Description: Production/live equipment

Tag: backup
  - Color: 2196f3 (Blue)
  - Description: Backup/standby equipment

Tag: end-of-life
  - Color: f44336 (Red)
  - Description: Equipment approaching end of life

Tag: critical
  - Color: 000000 (Black)
  - Description: Critical infrastructure
```

**Tags এর ব্যবহার:**

```
Device: R-DN-MIR-CORE-01
  - Tags: production, critical
  
Device: R-DN-MIR-CORE-02
  - Tags: backup
```

এখন filter করতে পারবেন: "Show me all devices with 'critical' tag"

#### Custom Fields - নিজের মতো ফিল্ড

Nautobot এ যদি কোনো built-in field না থাকে, custom field তৈরি করুন।

Nirvor Communication এর দরকার warranty tracking:

```
Custom Field: Warranty Expiry Date
  - Content Types: dcim.device
  - Type: Date
  - Label: Warranty Expiry Date
  - Description: Device warranty expiration date
  - Required: No
  - Default: (empty)
```

এখন প্রতিটা device এ এই field দেখাবে:

```
Device: R-DN-MIR-CORE-01
  - Warranty Expiry Date: 2027-01-15
```

আরেকটা উদাহরণ - Building tracking:

```
Custom Field: Building Name
  - Content Types: dcim.device
  - Type: Text
  - Label: Building Name
  - Description: Building where device is located
```

```
Device: SW-DN-MIR-ACC-01
  - Building Name: Building A, Sector 12
```

#### Custom Relationships (Nautobot 3.0 নতুন)

এটা একটা শক্তিশালী নতুন ফিচার। দুটো যেকোনো object এর মধ্যে custom relationship তৈরি করা যায়।

**উদাহরণ:** Device এবং Customer এর মধ্যে relationship

```
Relationship: Device Serves Customer
  - Source Type: dcim.device
  - Destination Type: tenancy.tenant
  - Type: Many to Many
  - Description: Which devices serve which customers
```

এখন একটা device এর সাথে multiple customers link করা যাবে:

```
Device: SW-DN-MIR-ACC-01
  - Serves Customers:
    * ABC Corporation
    * XYZ Limited
    * DEF Enterprises
```

---

### ডেটা মডেলের রিলেশনশিপ - সব কীভাবে যুক্ত

এখন দেখি কীভাবে সব একসাথে কাজ করে। Nirvor Communication এর Mirpur POP এর একটা সম্পূর্ণ উদাহরণ:

#### উদাহরণ: মিরপুর পপ এর সম্পূর্ণ ডেটা স্ট্রাকচার

**1. Location Hierarchy:**

```
Dhaka North Zone (Location Type: Zone)
  └── Mirpur Cluster (Location Type: Cluster)
       └── Mirpur POP (Location Type: POP)
            ├── Rack A (Location Type: Rack)
            └── Rack B (Location Type: Rack)
```

**2. Devices in Mirpur POP:**

```
R-DN-MIR-CORE-01
  - Type: MikroTik CCR2004
  - Role: Core Router
  - Location: Mirpur POP → Rack A → Position 25U
  - Status: Active
  - Tags: production, critical
  - Custom Fields:
    * Warranty Expiry: 2027-01-15
  
SW-DN-MIR-DIST-01
  - Type: TP-Link TL-SG3428
  - Role: Distribution Switch
  - Location: Mirpur POP → Rack A → Position 20U
  - Status: Active
  - Tags: production

SW-DN-MIR-ACC-01
  - Type: TP-Link TL-SG1024D
  - Role: Access Switch
  - Location: Mirpur POP → Rack B → Position 15U
  - Status: Active
  - Tags: production
  - Custom Fields:
    * Building Name: Building A, Sector 12
```

**3. Cables connecting them:**

```
Cable: CORE-TO-DIST-01
  - From: R-DN-MIR-CORE-01 → sfp-sfpplus2
  - To: SW-DN-MIR-DIST-01 → sfp1
  - Type: SMF
  - Length: 5m

Cable: DIST-TO-ACC-01
  - From: SW-DN-MIR-DIST-01 → ether1
  - To: SW-DN-MIR-ACC-01 → ether24
  - Type: Cat6
  - Length: 20m
```

**4. IP Addresses:**

```
10.10.1.1/32
  - Assigned to: R-DN-MIR-CORE-01 → lo0
  - Role: Loopback
  - DNS: r-mir-core-01.nirvor.bd

103.125.42.130/30
  - Assigned to: R-DN-MIR-CORE-01 → sfp-sfpplus1
  - Role: Network
  - DNS: r-mir-uplink.nirvor.bd
  - Description: BTCL uplink

10.10.10.11/24
  - Assigned to: SW-DN-MIR-DIST-01 → vlan10
  - VLAN: MGMT_MIRPUR
  - DNS: sw-mir-dist-01.nirvor.bd
```

**5. VLANs:**

```
VLAN 10 - MGMT_MIRPUR
  - Location: Mirpur POP
  - Prefixes: 10.10.10.0/24

VLAN 100 - RESIDENTIAL_MIR
  - Location: Mirpur POP
  - Prefixes: 103.125.40.0/24
```

**6. Circuit:**

```
Circuit: BTCL-MIR-001
  - Provider: BTCL
  - Type: Fiber Optic
  - Commit Rate: 5 Gbps
  - Termination Z:
    * Device: R-DN-MIR-CORE-01
    * Interface: sfp-sfpplus1
```

#### ডেটা ফ্লো ভিজুয়ালাইজেশন

```
BTCL Provider
    |
    | Circuit: BTCL-MIR-001 (5 Gbps)
    ↓
[R-DN-MIR-CORE-01] sfp-sfpplus1 (103.125.42.130/30)
    | lo0 (10.10.1.1/32)
    |
    | Cable: CORE-TO-DIST-01 (SMF, 5m)
    | sfp-sfpplus2 ↔ sfp1
    ↓
[SW-DN-MIR-DIST-01] vlan10 (10.10.10.11/24)
    |
    | Cable: DIST-TO-ACC-01 (Cat6, 20m)
    | ether1 ↔ ether24
    ↓
[SW-DN-MIR-ACC-01]
    | ether1-23: Customer connections
    |
    └→ Building A customers (VLAN 100)
```

এই পুরো structure Nautobot এর ডেটাবেসে store করা। এখন যদি কেউ জিজ্ঞাসা করে:

**প্রশ্ন:** "মিরপুর পপে মোট কয়টা ডিভাইস?"
**উত্তর:** Nautobot এ filter করুন Location = Mirpur POP

**প্রশ্ন:** "BTCL uplink কোন ডিভাইসের কোন পোর্টে?"
**উত্তর:** Circuit BTCL-MIR-001 এর Termination Z দেখুন

**প্রশ্ন:** "Building A এর কাস্টমাররা কোন সুইচে আছে?"
**উত্তর:** Custom Field "Building Name" = Building A filter করুন

**প্রশ্ন:** "কোর রাউটার থেকে এক্সেস সুইচ পর্যন্ত physical path কী?"
**উত্তর:** Cable Trace ফিচার ব্যবহার করুন

---

### ডেটা ইন্টিগ্রিটি - Nautobot কীভাবে নিশ্চিত করে

Nautobot অনেক validation করে যাতে ভুল ডেটা ঢুকতে না পারে।

#### Referential Integrity

যদি একটা Location মুছতে চান যেখানে Devices আছে, Nautobot বলবে:

```
Error: Cannot delete location "Mirpur POP" because:
  - 3 devices are assigned to this location
  - 2 racks are in this location

Delete or reassign these first.
```

#### Unique Constraints

একই নামে দুটো device তৈরি করা যাবে না:

```
Error: Device with name "R-DN-MIR-CORE-01" already exists.
```

একই IP address দুবার assign করা যাবে না:

```
Error: IP address "10.10.1.1/32" is already assigned to:
  - Device: R-DN-MIR-CORE-01
  - Interface: lo0
```

#### Status Validation

Offline device এ নতুন IP assign করতে গেলে warning:

```
Warning: This device status is "Offline". 
Are you sure you want to assign an IP?
```

#### Custom Validation

Custom validation rules তৈরি করা যায় plugins দিয়ে। যেমন:

```
Rule: Device নামে অবশ্যই site code থাকতে হবে
  - Valid: R-DN-MIR-CORE-01 (DN আছে)
  - Invalid: R-MIR-CORE-01 (DN নেই)
```

---

### GraphQL API - Advanced Querying

Nautobot 3.0 এ GraphQL API আছে যা REST API থেকে শক্তিশালী।

**REST API তে:**

Mirpur POP এর সব device এবং তাদের IP addresses নিতে:

```
GET /api/dcim/devices/?location=mirpur-pop
  - Response: Devices list
  
Then for each device:
GET /api/ipam/ip-addresses/?device=<device-id>
  - Response: IPs for that device
```

অনেকগুলো request লাগে।

**GraphQL এ:**

একটা request এ সব:

```graphql
query {
  devices(location: "mirpur-pop") {
    name
    device_type {
      model
    }
    interfaces {
      name
      ip_addresses {
        address
        dns_name
      }
    }
  }
}
```

Response এ পুরো nested data একসাথে আসবে। অনেক দ্রুত।

---

Nirvor Communication এখন বুঝে গেছে তাদের নেটওয়ার্ক কীভাবে Nautobot এ রিপ্রেজেন্ট হবে। পরের চ্যাপ্টারে আমরা দেখব কীভাবে UI দিয়ে actual ডেটা এন্ট্রি করতে হয়। এভাবে একটু একটু করে তারা পুরো নেটওয়ার্ক Nautobot-এ ডকুমেন্ট করবে। শুরুতে সময় লাগবে, কিন্তু একবার হয়ে গেলে পরে সব সহজ হয়ে যায়। পাশাপাশি, থিওরি থেকে প্র্যাকটিসে পুরোপুরি নামা যাবে।
