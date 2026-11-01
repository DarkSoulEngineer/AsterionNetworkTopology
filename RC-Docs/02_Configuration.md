# Asterion Digital Services
## Document tehnic: configurația echipamentelor și serviciilor

Acest document descrie configurația implementată și comenzile de referință necesare pentru reconstruirea infrastructurii. Blocurile CLI se execută pe dispozitivul indicat, nu pe PC-PT. Valorile pentru care există rezultate CLI sunt identificate în tabelele de verificare. Pentru unele funcții, fișierul `.pkt` rămâne sursa de adevăr pentru configurația efectivă.

**Stări folosite în document:**

- **Confirmat:** susținut de rezultate CLI sau teste de conectivitate din proiect.
- **Documentat:** prezent în configurațiile de lucru, dar fără export complet și recent al `running-config`.
- **De verificat:** necesită consultarea proiectului `.pkt`, în special porturi, configurații de serviciu sau politici NAT suplimentare.

Parolele sunt reprezentate prin marcatori precum `<PAROLA_ADMIN>` și `<PAROLA_FTP>`. Se înlocuiesc local înainte de aplicare și nu se publică în documentația de predare. Comenzile `crypto key generate rsa` sunt interactive; cheia nu se generează din nou dacă SSH funcționează deja.

## 2. Planul de adresare

### 2.1. Subrețele

| Segment | Rețea | Mască | Gateway/router |
| :--- | :--- | :--- | :--- |
| Test extern | `192.168.100.0/24` | `255.255.255.0` | `192.168.100.1` |
| External–ASA | `10.0.0.0/30` | `255.255.255.252` | `10.0.0.1` / `10.0.0.2` |
| Tranzit intern | `10.0.1.0/24` | `255.255.255.0` | ASA `10.0.1.1` |
| Floor1 | `192.168.10.0/24` | `255.255.255.0` | `192.168.10.1` |
| Floor2 | `192.168.20.0/24` | `255.255.255.0` | `192.168.20.1` |
| Floor3 | `192.168.30.0/24` | `255.255.255.0` | `192.168.30.1` |
| ClientData | `192.168.40.0/24` | `255.255.255.0` | `192.168.40.1` |

### 2.2. Inventar IP pentru infrastructură

| Dispozitiv | Interfață | Adresă IPv4 |
| :--- | :--- | :--- |
| External | `Gi0/0/0` | `192.168.100.1/24` |
| External | `Gi0/0/1` | `10.0.0.1/30` |
| ASA | `Gi1/1 outside` | `10.0.0.2/30` |
| ASA | `Gi1/2 inside` | `10.0.1.1/24` |
| Floor1 | `Gi0/0` / `Vlan1` | `10.0.1.11/24` / `192.168.10.1/24` |
| Floor2 | `Gi0/0` / `Vlan1` | `10.0.1.12/24` / `192.168.20.1/24` |
| Floor3 | `Gi0/0` / `Vlan1` / `Gi0/1` | `10.0.1.13/24` / `192.168.30.1/24` / `192.168.40.1/24` |
| Switch1 | `Vlan1` | `192.168.40.11/24` |
| Switch2 | `Vlan1` | `192.168.40.12/24` |
| Switch3 | `Vlan1` | `192.168.40.13/24` |
| SNMP-Monitor | `Fa0` | `10.0.1.20/24` |
| ASA Apache NAT | adresă tradusă | `10.0.0.5` |

## 3. Conexiuni fizice

### 3.1. Perimetru și distribuție

| Origine:port | Destinație:port | Funcție |
| :--- | :--- | :--- |
| Attacker `Fa0` | External `Gi0/0/0` | Rețea de test extern |
| External `Gi0/0/1` | ASA `Gi1/1` | Outside |
| ASA `Gi1/2` | Entry `Fa0/1` | Inside |
| Entry `Fa0/2` | Floor1 `Gi0/0` | Tranzit |
| Entry `Fa0/3` | Floor2 `Gi0/0` | Tranzit |
| Entry `Fa0/4` | Floor3 `Gi0/0` | Tranzit |
| Entry `Fa0/5` | SNMP-Monitor `Fa0` | Monitorizare |
| Floor3 `Gi0/1` | Switch1 `Gi0/1` | Acces ClientData |

### 3.2. Module EtherSwitch ale routerelor 2901

| Router | Port | Echipament local |
| :--- | :--- | :--- |
| Floor1 | `Fa0/0/0` | Apache |
| Floor1 | `Fa0/0/1` | Corporate PC |
| Floor1 | `Fa0/0/2` | FTP |
| Floor1 | `Fa0/0/3` | Floor1Printer |
| Floor2 | `Fa0/0/0` | Red PC |
| Floor2 | `Fa0/0/1` | Vulnerable PC |
| Floor2 | `Fa0/0/2` | VulnerableServer |
| Floor2 | `Fa0/0/3` | Floor2Printer |
| Floor3 | `Fa0/0/0` | Admin PC |
| Floor3 | `Fa0/0/1` | SysLog |
| Floor3 | `Fa0/0/2` | Floor3Printer |

Acestea sunt mapările din configurația inițială. Verificarea fizică finală se face prin selectarea cablurilor din proiectul `.pkt`. Modulele EtherSwitch folosesc interfața logică `Vlan1` pentru gateway și nu necesită comenzi IP pe fiecare port FastEthernet.

### 3.3. Switch-uri redundante

| Conexiune | Interfață A | Interfață B |
| :--- | :--- | :--- |
| Switch1–Switch2 | `Fa0/1` | `Fa0/1` |
| Switch1–Switch2 | `Fa0/4` | `Fa0/4` |
| Switch1–Switch3 | `Fa0/3` | `Fa0/3` |
| Switch1–Switch3 | `Fa0/6` | `Fa0/6` |
| Switch2–Switch3 | două legături | identificarea porturilor prin CDP |
| Switch2–ClientData | `Fa0/24` | `Fa0` server |
| Switch3–PC0 | `Fa0/24` | `Fa0` PC |

## 4. Router External (ISR4331)

**Rol:** conectivitate către rețeaua Attacker și transportul traficului către firewall.

```cisco
 enable
 configure terminal
 hostname External
 no ip domain-lookup
 interface GigabitEthernet0/0/0
  description TO_ATTACKER
  ip address 192.168.100.1 255.255.255.0
  no shutdown
 exit
 interface GigabitEthernet0/0/1
  description TO_ASA_OUTSIDE
  ip address 10.0.0.1 255.255.255.252
  no shutdown
 exit
 ip route 10.0.0.5 255.255.255.255 10.0.0.2
 end
 copy running-config startup-config
```

**Verificare:**

```cisco
show ip interface brief
show ip route
ping 10.0.0.2
```

Ruta host `10.0.0.5/32` către `10.0.0.2` este documentată în tabela de rutare a routerului extern.

## 5. Switch Entry (2960-24TT)

**Rol:** interconectarea ASA, Floor1–3 și SNMP-Monitor în segmentul `10.0.1.0/24`.

```cisco
enable
configure terminal
hostname Entry
no ip domain-lookup
interface FastEthernet0/1
 description TO_ASA_FIREWALL
 switchport mode access
 switchport access vlan 1
 no shutdown
exit
interface FastEthernet0/2
 description TO_FLOOR1
 switchport mode access
 switchport access vlan 1
 no shutdown
exit
interface FastEthernet0/3
 description TO_FLOOR2
 switchport mode access
 switchport access vlan 1
 no shutdown
exit
interface FastEthernet0/4
 description TO_FLOOR3
 switchport mode access
 switchport access vlan 1
 no shutdown
exit
interface FastEthernet0/5
 description SNMP_MONITOR
 switchport mode access
 switchport access vlan 1
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
exit
interface range FastEthernet0/10 - 13
 switchport mode access
 switchport access vlan 25
exit
interface FastEthernet0/24
 switchport mode access
 switchport access vlan 99
exit
end
copy running-config startup-config
```

Fa0/10–Fa0/13 în VLAN 25 și Fa0/24 în VLAN 99 provin din `show running-config` pe Entry; nu sunt utilizate de legăturile active spre routere. Pe Entry rămâne activ PVST implicit (conform configurației observate), separat de grupul inferior Rapid PVST+.

**Verificare:**

```cisco
show interfaces status
show vlan brief
show mac address-table
show spanning-tree vlan 1
```

## 6. Router Floor1 (Cisco 2901)

**Rol:** gateway Floor1, acces Corporate, HTTP și FTP; administrare SSH și agent SNMP; export Syslog.

### 6.1. Adresare și rutare

```cisco
enable
configure terminal
hostname Floor1
no ip domain-lookup
interface GigabitEthernet0/0
 description TO_ENTRY
 ip address 10.0.1.11 255.255.255.0
 no shutdown
exit
interface Vlan1
 description FLOOR1_LAN
 ip address 192.168.10.1 255.255.255.0
 no shutdown
exit
ip route 0.0.0.0 0.0.0.0 10.0.1.1
ip route 192.168.20.0 255.255.255.0 10.0.1.12
ip route 192.168.30.0 255.255.255.0 10.0.1.13
end
copy running-config startup-config
```

Porturile `Fa0/0/0–3` aparțineau modulului EtherSwitch și au fost observate Up/Up. Dacă routerul nu acceptă `switchport` pe aceste interfețe, nu se introduce configurația de switch pe `Gi0/0`; se verifică apartenența la VLAN prin `show vlan-switch brief` și se păstrează modulul existent.

### 6.2. SSH

```cisco
configure terminal
ip domain-name enterprise.local
username admin privilege 15 secret <PAROLA_ADMIN>
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
line vty 0 4
 login local
 transport input ssh
 exec-timeout 10 0
exit
line vty 5 15
 login local
 transport input ssh
 exec-timeout 10 0
exit
end
copy running-config startup-config
```

**Generarea inițială RSA** (o singură dată, doar dacă lipsesc cheile): `crypto key generate rsa`, cu modul `1024` în simulator. Nu suprascrie parola/cheile deja funcționale.

### 6.3. SNMP și Syslog

```cisco
configure terminal
snmp-server community RC_MONITOR RO
service timestamps log datetime msec
logging on
logging host 192.168.30.20
end
copy running-config startup-config
```

Varianta cu ACL pe comanda `snmp-server community` și comenzile `snmp-server location/contact` au fost respinse de IOS în laborator; se folosește sintaxa simplificată de mai sus.

**Verificare Floor1:**

```cisco
show ip interface brief
show ip route
show ip ssh
show ssh
show running-config | include snmp
show logging
```

## 7. Router Floor2 (Cisco 2901)

**Rol:** gateway pentru laboratorul de evaluare a securității și acces la serverul de test.

### 7.1. Adresare și rutare

```cisco
enable
configure terminal
hostname Floor2
no ip domain-lookup
interface GigabitEthernet0/0
 description TO_ENTRY
 ip address 10.0.1.12 255.255.255.0
 no shutdown
exit
interface Vlan1
 description FLOOR2_LAN
 ip address 192.168.20.1 255.255.255.0
 no shutdown
exit
ip route 0.0.0.0 0.0.0.0 10.0.1.1
ip route 192.168.10.0 255.255.255.0 10.0.1.11
ip route 192.168.30.0 255.255.255.0 10.0.1.13
end
copy running-config startup-config
```

### 7.2. SSH

```cisco
configure terminal
ip domain-name enterprise.local
username admin privilege 15 secret <PAROLA_ADMIN>
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
line vty 0 4
 login local
 transport input ssh
 exec-timeout 10 0
exit
line vty 5 15
 login local
 transport input ssh
 exec-timeout 10 0
exit
end
copy running-config startup-config
```

Pentru chei RSA lipsă: `crypto key generate rsa` (modul `1024` în Packet Tracer).

### 7.3. SNMP și Syslog

```cisco
configure terminal
snmp-server community RC_MONITOR RO
service timestamps log datetime msec
logging on
logging host 192.168.30.20
end
copy running-config startup-config
```

**Verificare Floor2:**

```cisco
show ip interface brief
show ip route
show ip ssh
show ssh
show running-config | section line vty
show running-config | include snmp
show logging
```

**Dovadă SSH existentă:** SSH v2 activ, `admin`, `aes128-cbc`, `hmac-sha1`, `Session Started`, `transport input ssh` pe VTY.

## 8. Router Floor3 (Cisco 2901)

**Rol:** gateway administrare, Syslog și router de acces către rețeaua redundantă ClientData.

### 8.1. Adresare și rutare

```cisco
enable
configure terminal
hostname Floor3
no ip domain-lookup
interface GigabitEthernet0/0
 description TO_ENTRY
 ip address 10.0.1.13 255.255.255.0
 no shutdown
exit
interface Vlan1
 description FLOOR3_LAN
 ip address 192.168.30.1 255.255.255.0
 no shutdown
exit
interface GigabitEthernet0/1
 description LOWER_LOCAL_NETWORK
 ip address 192.168.40.1 255.255.255.0
 no shutdown
exit
ip route 0.0.0.0 0.0.0.0 10.0.1.1
ip route 192.168.10.0 255.255.255.0 10.0.1.11
ip route 192.168.20.0 255.255.255.0 10.0.1.12
end
copy running-config startup-config
```

**Confirmat prin CDP:** Floor3 `Gi0/0` ↔ Entry `Fa0/4`; Floor3 `Gi0/1` ↔ Switch1 `Gi0/1`. Interfețele au fost observate Up/Up, iar rețeaua `192.168.40.0/24` figurează direct conectată.

### 8.2. SSH

```cisco
configure terminal
ip domain-name enterprise.local
username admin privilege 15 secret <PAROLA_ADMIN>
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
line vty 0 4
 login local
 transport input ssh
 exec-timeout 10 0
exit
line vty 5 15
 login local
 transport input ssh
 exec-timeout 10 0
exit
end
copy running-config startup-config
```

Cheile RSA, dacă lipsesc, se generează interactiv. Testul direct SSH pe Floor3 se validează separat.

### 8.3. SNMP și Syslog

```cisco
configure terminal
snmp-server community RC_MONITOR RO
service timestamps log datetime msec
logging on
logging host 192.168.30.20
end
copy running-config startup-config
```

**Verificare Floor3:**

```cisco
show ip interface brief
show cdp neighbors
show ip route connected
show ip ssh
show running-config | include snmp
show logging
```

## 9. Grupul de switch-uri redundante (2960)

### 9.1. Switch1: root bridge

```cisco
enable
configure terminal
hostname Switch1
spanning-tree mode rapid-pvst
spanning-tree vlan 1 root primary
interface Vlan1
 description MANAGEMENT
 ip address 192.168.40.11 255.255.255.0
 no shutdown
exit
ip default-gateway 192.168.40.1
end
copy running-config startup-config
```

**Rezultat observat:** `This bridge is the root`, prioritate efectivă `24577` pentru VLAN 1. Conexiuni către Switch2 prin Fa0/1 și Fa0/4, către Switch3 prin Fa0/3 și Fa0/6; către Floor3 prin Gi0/1.

### 9.2. Switch2: root secundar și ClientData

```cisco
enable
configure terminal
hostname Switch2
spanning-tree mode rapid-pvst
spanning-tree vlan 1 root secondary
interface Vlan1
 description MANAGEMENT
 ip address 192.168.40.12 255.255.255.0
 no shutdown
exit
ip default-gateway 192.168.40.1
interface FastEthernet0/24
 description CLIENT_DATA_SERVER
 switchport mode access
 switchport access vlan 1
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
exit
end
copy running-config startup-config
```

**Rezultat observat:** prioritate configurată `28672`, `Fa0/1 Root FWD`, `Fa0/4 Altn BLK`, `Fa0/24 Desg FWD`. `Fa0/24` are PortFast și BPDU Guard în `startup-config`.

### 9.3. Switch3: client PC0

```cisco
enable
configure terminal
hostname Switch3
spanning-tree mode rapid-pvst
interface Vlan1
 description MANAGEMENT
 ip address 192.168.40.13 255.255.255.0
 no shutdown
exit
ip default-gateway 192.168.40.1
interface FastEthernet0/24
 description PC0_ACCESS
 switchport mode access
 switchport access vlan 1
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
exit
end
copy running-config startup-config
```

**Rezultat observat:** `Fa0/3 Root FWD`, `Fa0/6 Altn BLK`, `Fa0/2 Altn BLK`, `Fa0/5 Altn BLK`, `Fa0/24 Desg FWD`. Portul Fa0/24 a fost văzut fizic Up/Up. Nu se aplică PortFast interconexiunilor dintre switch-uri.

### 9.4. Verificare STP și management

```cisco
show spanning-tree vlan 1
show spanning-tree summary
show interfaces status
show cdp neighbors
show ip interface brief
show mac address-table
```

În testul de redundanță s-a dezactivat temporar `Switch3 Fa0/3`, apoi s-a reactivat cu `no shutdown`. Testele PC0 → gateway și ClientData sunt documentate separat.

## 10. Firewall ASA 5506-X

**Rol:** separarea segmentului extern de rețelele interne, rutare perimetrală, controlul accesului și static NAT pentru HTTP.

### 10.1. Interfețe și rute

```cisco
enable
configure terminal
hostname Firewall
interface GigabitEthernet1/1
 nameif outside
 security-level 0
 ip address 10.0.0.2 255.255.255.252
 no shutdown
exit
interface GigabitEthernet1/2
 nameif inside
 security-level 100
 ip address 10.0.1.1 255.255.255.0
 no shutdown
exit
route outside 0.0.0.0 0.0.0.0 10.0.0.1
route inside 192.168.10.0 255.255.255.0 10.0.1.11
route inside 192.168.20.0 255.255.255.0 10.0.1.12
route inside 192.168.30.0 255.255.255.0 10.0.1.13
end
write memory
```

### 10.2. NAT static și ACL HTTP

```cisco
configure terminal
object network APACHE-WEB
 host 192.168.10.10
 nat (inside,outside) static 10.0.0.5
exit
access-list APACHE_ONLY extended permit tcp host 192.168.100.10 host 192.168.10.10 eq www
access-group APACHE_ONLY in interface outside
end
write memory
```

Adresa publicată `10.0.0.5` este accesată din segmentul extern de test și rutată către ASA prin ruta host de pe External. În notițele proiectului, `show xlate` indică traducerea `inside:192.168.10.10/32` → `outside:10.0.0.5/32`. Accesul HTTP extern este consemnat ca reușit în tabelul final de verificare; pentru documentația de predare se atașează captura browserului și contoarele actuale NAT.

### 10.3. Verificare ASA

```cisco
show interface ip brief
show route
show nat
show xlate
show access-list
show running-config | include access-group
```

**De verificat din `.pkt`:** orice regulă suplimentară NAT/PAT pentru trafic inițiat de clienții interni către segmentul extern. Comenzile de mai sus descriu regula statică documentată; ele nu reprezintă configurația completă a întregului `running-config` ASA.

## 11. PC-uri și imprimante: adresare statică

Pentru PC-PT, setările se aplică în **Desktop → IP Configuration → Static**. Pentru Printer-PT se folosesc câmpurile IP din **Config**. Toate dispozitivele acestei secțiuni folosesc masca `255.255.255.0`.

| Dispozitiv | IP | Gateway | Conectare |
| :--- | :--- | :--- | :--- |
| Attacker PC | `192.168.100.10` | `192.168.100.1` | External |
| Corporate PC | `192.168.10.20` | `192.168.10.1` | Floor1 |
| Red PC | `192.168.20.10` | `192.168.20.1` | Floor2 |
| Vulnerable PC | `192.168.20.20` | `192.168.20.1` | Floor2 |
| Admin PC | `192.168.30.10` | `192.168.30.1` | Floor3 |
| PC0 | `192.168.40.10` | `192.168.40.1` | Switch3 |
| SNMP-Monitor | `10.0.1.20` | nesolicitat pentru `10.0.1.0/24` | Entry |
| Floor1Printer | `192.168.10.40` | `192.168.10.1` | Floor1 |
| Floor2Printer | `192.168.20.40` | `192.168.20.1` | Floor2 |
| Floor3Printer | `192.168.30.30` | `192.168.30.1` | Floor3 |

**Teste rapide de pe PC-PT:** `ipconfig /all`, `ping <gateway>`, `arp -a`.

## 12. Serverele și serviciile

Toate serverele folosesc masca `255.255.255.0` și adresare statică.

### 12.1. Server Apache / HTTP

| Câmp | Valoare |
| :--- | :--- |
| IP | `192.168.10.10` |
| Gateway | `192.168.10.1` |
| Conectare | Floor1 |
| Serviciu | `Services → HTTP → On` |
| Acces intern | `http://192.168.10.10` |
| Acces extern prin NAT | `http://10.0.0.5` |

În Packet Tracer, dispozitivul Server-PT emulează un server web, etichetat „Apache” în topologie.

### 12.2. Server FTP

| Câmp | Valoare |
| :--- | :--- |
| IP | `192.168.10.30` |
| Gateway | `192.168.10.1` |
| Conectare | Floor1 |
| Serviciu | `Services → FTP → On` |

Se configurează un cont FTP în interfața serviciului, cu parolă aleasă local. Test: `ftp 192.168.10.30`. Numele concret al contului nu este prezent în configurația furnizată.

### 12.3. VulnerableServer

| Câmp | Valoare |
| :--- | :--- |
| IP | `192.168.20.30` |
| Gateway | `192.168.20.1` |
| Conectare | Floor2 |
| Serviciu | `Services → HTTP → On` |
| URL test | `http://192.168.20.30` |

### 12.4. Server SysLog

| Câmp | Valoare |
| :--- | :--- |
| IP | `192.168.30.20` |
| Gateway | `192.168.30.1` |
| Conectare | Floor3 |
| Serviciu | `Services → SYSLOG → On` |
| Port | UDP 514 |

Mesajele din Floor1, Floor2 și Floor3 au fost observate în interfața `Services → SYSLOG`, inclusiv evenimente de stare și configurare.

### 12.5. Server ClientData

| Câmp | Valoare |
| :--- | :--- |
| IP | `192.168.40.20` |
| Gateway | `192.168.40.1` |
| Conectare | Switch2 Fa0/24 |
| Serviciu specific | De verificat în `Services` dacă se dorește un serviciu de fișiere |

Conectivitatea cu PC0 și Switch3 a fost validată prin ICMP. Nu se presupune o configurație de fișiere sau autentificare care nu a fost furnizată.

## 13. Stația SNMP-Monitor: MIB Browser

**Echipament:** PC-PT, `10.0.1.20/24`, conectat la Entry Fa0/5.

**Aplicație:** Desktop → MIB Browser → Advanced.

| Câmp | Valoare |
| :--- | :--- |
| Address | `10.0.1.11` pentru Floor1 |
| Port | `161` |
| SNMP Version | `v2` |
| Read Community | `RC_MONITOR` |
| Operație | `Get` |
| OID uptime | `1.3.6.1.2.1.1.3.0` |
| OID nume | `1.3.6.1.2.1.1.5.0` |

Pentru Floor2 și Floor3 se schimbă Address în `10.0.1.12` sau `10.0.1.13`. Selectarea OID din MIB Tree funcționează în Packet Tracer. Captura `sysUpTime` pentru Floor1 a returnat o valoare de `2 hours 2 minutes 5 seconds` (tip `TimeTicks`), iar capturile de `sysName` sunt documentate pentru Floor2 și Floor3.

## 14. Protocolul SSH: testarea administrării

Din PC-PT Command Prompt:

```text
ssh -l admin 10.0.1.11
ssh -l admin 10.0.1.12
ssh -l admin 10.0.1.13
```

De pe PC0 se poate testa accesul local spre Floor3 prin `ssh -l admin 192.168.40.1`. Testele SSH pe Floor1 și Floor2 au fost raportate reușite; pe Floor2 s-au văzut `SSH Enabled - version 2.0`, `Session Started`, algoritmii `aes128-cbc` și `hmac-sha1`.

**Comenzi de evidență:**

```cisco
show ip ssh
show ssh
show running-config | section line vty
```

## 15. Verificări finale, pe dispozitiv

| Echipament | Comenzi / test |
| :--- | :--- |
| External | `show ip route`, `show ip interface brief` |
| Entry | `show interfaces status`, `show vlan brief` |
| Floor1 | `show ip route`, `show ip ssh`, `show logging` |
| Floor2 | `show ip route`, `show ssh`, `show logging` |
| Floor3 | `show cdp neighbors`, `show ip route connected`, `show logging` |
| Switch1 | `show spanning-tree vlan 1`, `show cdp neighbors` |
| Switch2 | `show spanning-tree vlan 1`, `show running-config` |
| Switch3 | `show spanning-tree vlan 1`, `show interfaces fa0/24` |
| ASA | `show nat`, `show xlate`, `show access-list`, `show route` |
| Corporate PC | ping gateway, HTTP Apache, FTP |
| PC0 | ping gateway, ClientData, Switch1–3 |
| SNMP-Monitor | ping Floor1–3, MIB Browser GET |
| SysLog | `Services → SYSLOG` |
| Attacker PC | Web Browser către `http://10.0.0.5` |

### 15.1. Test de redundanță STP

1. Se înregistrează starea `show spanning-tree vlan 1` pe Switch1–3.
2. Se dezactivează temporar `Switch3 Fa0/3`, legătura root curentă către Switch1.
3. Se inspectează noua stare STP și se testează PC0 → `192.168.40.1` și PC0 → `192.168.40.20`.
4. Se reactivează `Switch3 Fa0/3` și se verifică revenirea porturilor în starea normală.

Configurația nu se salvează în timpul testului cu portul dezactivat; se salvează la final, după restaurarea sa.

## 16. Controlul configurației și inventarul de final

Rularea `copy running-config startup-config` este necesară pe routere și switch-uri; pe ASA se folosește `write memory`. Înainte de arhivarea proiectului, se exportă `show running-config` individual de pe fiecare echipament pentru a conserva configurația exactă a fișierului `.pkt`.

**Elemente cu verificare finală în Packet Tracer:**

- Porturile exacte ale celor două legături Switch2–Switch3.
- Conturile și permisiunile serviciului FTP.
- Serviciile suplimentare activate pe ClientData.
- Eventualele reguli NAT/PAT de ieșire prezente pe ASA.
- Testul SSH efectiv pe Floor3 și porturile VTY disponibile.
- Configurația IPv6, dacă a fost adăugată după documentația analizată.
