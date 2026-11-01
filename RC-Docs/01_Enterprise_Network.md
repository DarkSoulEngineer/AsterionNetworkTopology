# Asterion Digital Services

## Proiectarea și evaluarea infrastructurii de rețea enterprise

**Disciplina:** Rețele de Calculatoare  
**Platformă de implementare:** Cisco Packet Tracer  
**Arhitectură:** infrastructură distribuită pe trei etaje, nucleu de interconectare, securitate perimetrală și servicii de administrare centralizată  
**Teme de implementare:** Spanning Tree Protocol (STP), Secure Shell (SSH), Simple Network Management Protocol (SNMP), Network Address Translation (NAT)

## 1. Prezentarea proiectului

Asterion Digital Services este o companie specializată în dezvoltarea și operarea serviciilor web, administrarea infrastructurilor IT și evaluarea securității sistemelor informatice. Activitatea organizației este distribuită între departamente cu responsabilități distincte: operațiuni corporative, laborator de securitate și administrare tehnică.

Infrastructura de comunicații deservește o clădire de birouri cu trei etaje. Fiecare nivel dispune de o rețea locală proprie, iar comunicația dintre acestea este asigurată printr-un segment central de interconectare. Un firewall amplasat la perimetru controlează accesul către mediul extern. Serviciile de administrare și monitorizare permit echipei IT să urmărească funcționarea echipamentelor și să investigheze evenimentele de rețea.

Proiectul urmărește realizarea unei infrastructuri coerente din punct de vedere operațional, în care serviciile de producție, activitățile de securitate și managementul rețelei sunt organizate în zone cu roluri bine definite.

### 1.1. Obiective de business

| Obiectiv | Necesitate operațională | Soluție utilizată |
|---|---|---|
| Disponibilitatea serviciilor | Acces constant la aplicația web, transfer de fișiere și resursele interne | Rețele locale conectate prin routerele de etaj |
| Delimitarea activităților | Separarea organizatorică a operațiunilor corporative, laboratorului de securitate și administrației | Subrețele IPv4 distincte pentru fiecare etaj |
| Controlul accesului extern | Publicarea selectivă a serviciului web și gestionarea traficului la perimetru | Cisco ASA, politici ACL și NAT |
| Administrarea infrastructurii | Acces la distanță pentru verificări și operațiuni de mentenanță | SSH pe routerele de etaj |
| Vizibilitate operațională | Identificarea dispozitivelor și observarea parametrilor de funcționare | SNMP și colectare centralizată Syslog |
| Continuitatea accesului la date | Menținerea conectivității în segmentul local dedicat ClientData | Legături Ethernet redundante controlate prin Rapid PVST+ |

### 1.2. Scenarii de utilizare

**Accesarea portalului de servicii.** Departamentul corporativ utilizează aplicația HTTP găzduită la primul etaj. Accesul din segmentul extern de test este gestionat prin firewall și mecanismul NAT, care permite asocierea serverului intern unei adrese de acces externe.

**Evaluarea securității.** Laboratorul de la etajul al doilea conține stații și un server destinate verificărilor controlate ale comunicațiilor. Aceste resurse sunt grupate într-o subrețea separată de celelalte departamente.

**Administrarea și investigarea evenimentelor.** Personalul IT utilizează SSH pentru conectarea la routere, SNMP pentru consultarea parametrilor de funcționare și Syslog pentru vizualizarea mesajelor generate de echipamente.

**Accesul continuu la date.** Segmentul ClientData de la etajul al treilea include trei switch-uri conectate redundant. Protocolul Rapid PVST+ selectează căile de forwarding și permite recuperarea conectivității după întreruperea unei legături active.

## 2. Arhitectura generală

### 2.1. Topologie fizică

![[NetworkTopology.png]]

![[NetworkTopologyMemaid.png]]

*Figura 1. Topologia infrastructurii implementate în Cisco Packet Tracer.

![[NetworkTopologyCisco.png]]

*Figura 2. Organizarea conexiunilor și a segmentelor funcționale.*

Rețeaua are un model de conectare centralizat în jurul switch-ului **Entry (Cisco 2960-24TT)**. Acesta interconectează firewall-ul, routerele Floor1–Floor3 și stația SNMP-Monitor.

Routerele Cisco 2901 asigură legătura dintre rețeaua de tranzit și subrețelele locale. Conectarea stațiilor, serverelor și imprimantelor de pe fiecare etaj se face prin modulele EtherSwitch integrate în routere. La etajul al treilea, infrastructura este extinsă cu un segment de acces format din Switch1, Switch2 și Switch3.

### 2.2. Segmentele arhitecturii

| Segment | Subrețea | Rol |
|---|---|---|
| Rețea externă de test | `192.168.100.0/24` | Simularea accesului din exterior |
| Legătură External–ASA | `10.0.0.0/30` | Interconectarea routerului extern cu firewall-ul |
| Rețea internă de tranzit | `10.0.1.0/24` | Interconectarea celor trei routere și accesul stației de monitorizare |
| Floor1 | `192.168.10.0/24` | Operațiuni corporative și servicii HTTP/FTP |
| Floor2 | `192.168.20.0/24` | Laborator de securitate |
| Floor3 | `192.168.30.0/24` | Administrare și Syslog |
| ClientData | `192.168.40.0/24` | Rețea de acces redundantă și managementul switch-urilor locale |

Planul de adresare folosește subrețele private distincte, cu un gateway dedicat fiecărui segment local. Legătura dintre routerul extern și firewall este alocată separat, printr-un prefix /30, adecvat unei interconexiuni cu două echipamente.

### 2.3. Fluxurile principale de comunicație

**Trafic intern între etaje:** stație locală → routerul etajului → Entry → routerul destinație → serviciul solicitat.

**Trafic perimetral:** segment extern → router External → ASA → Entry → routerul etajului → serviciul intern publicat.

**Administrare și monitorizare:** SNMP-Monitor → Entry → routerele de etaj; stațiile autorizate se conectează prin SSH, iar routerele expediază mesaje către serverul Syslog de la Floor3.

**Acces ClientData:** PC0 → Switch3 → infrastructura redundantă Switch1–Switch3 → Switch2 → serverul ClientData.

## 3. Organizarea departamentelor

### 3.1. Floor1: Corporate & Services

Primul etaj concentrează activitățile corporative și serviciile utilizate de organizație. Include o stație Corporate, un server HTTP denumit Apache, un server FTP și o imprimantă de departament.

Serverul HTTP reprezintă punctul de acces la portalul de servicii. Serverul FTP este inclus în scenariul de transfer de fișiere. Routerul Floor1 oferă gateway-ul subrețelei și comunică cu restul infrastructurii prin Entry.

### 3.2. Floor2: Security Laboratory

Etajul al doilea găzduiește laboratorul de evaluare a securității. Stația Red este utilizată în scenarii de testare, iar stația Vulnerable și serverul VulnerableServer reprezintă sistemele țintă din mediul de laborator. O imprimantă completează infrastructura locală.

Organizarea într-o subrețea dedicată permite definirea explicită a fluxurilor de comunicație asociate testelor și păstrează o delimitare clară a rolurilor în arhitectura de ansamblu.

### 3.3. Floor3: IT Administration

Etajul al treilea reunește stația Admin, serverul Syslog și infrastructura de acces către segmentul ClientData. Personalul tehnic utilizează această zonă pentru operațiuni de administrare și supravegherea serviciilor de rețea.

Routerul Floor3 deservește atât LAN-ul administrativ, cât și subrețeaua `192.168.40.0/24`, conectată printr-o interfață dedicată la Switch1.

### 3.4. Segmentul ClientData

Segmentul ClientData conține o stație de lucru, un server de date și trei switch-uri Cisco 2960 interconectate prin legături redundante. Switch1 este desemnat root bridge, iar Switch2 reprezintă candidatul secundar pentru această funcție.

În condiții normale, protocolul STP păstrează căile necesare forwarding-ului și menține celelalte conexiuni ca alternative. La pierderea unei legături active, infrastructura poate utiliza o cale disponibilă pentru a menține comunicația dintre stație și server.

## 4. Componente hardware și servicii

| Categorie | Model / tip | Rol în infrastructură |
|---|---|---|
| Routere de etaj | 3 × Cisco 2901 | Gateway-uri pentru subrețelele locale și rutare IPv4 |
| Router de perimetru | Cisco ISR4331 | Reprezentarea segmentului extern |
| Firewall | Cisco ASA 5506-X | Filtrare la perimetru și NAT |
| Switch central | Cisco 2960-24TT Entry | Interconectarea echipamentelor principale |
| Switch-uri redundante | 3 × Cisco 2960-24TT | Continuitatea conectivității în segmentul ClientData |
| Module EtherSwitch | Integrate în routerele Cisco 2901 | Conectarea dispozitivelor locale pe fiecare etaj |
| Stații de lucru | PC-PT | Operațiuni, administrare, testare și monitorizare |
| Servere | Server-PT | HTTP, FTP, laborator, Syslog și ClientData |
| Imprimante | Printer-PT | Servicii locale pe fiecare etaj |

Echipamentele au fost selectate pentru a reprezenta funcții distincte de rutare, switching, control al accesului, furnizare de servicii și monitorizare. Modulele EtherSwitch din routerele de etaj permit conectarea terminalelor locale fără introducerea unui switch dedicat suplimentar pentru fiecare LAN de bază.

## 5. Adresare, VLAN-uri și rutare

### 5.1. Plan de adresare

Subrețelele etajelor sunt alocate distinct, în clase de adrese private. Această organizare simplifică identificarea destinațiilor, configurarea gateway-urilor și documentarea fluxurilor dintre departamente.

| Subrețea | Gateway / punct de referință | Utilizare |
|---|---|---|
| `192.168.10.0/24` | `192.168.10.1` | Floor1 |
| `192.168.20.0/24` | `192.168.20.1` | Floor2 |
| `192.168.30.0/24` | `192.168.30.1` | Floor3 |
| `192.168.40.0/24` | `192.168.40.1` | ClientData și managementul celor trei switch-uri |
| `10.0.1.0/24` | ASA `10.0.1.1` | Tranzit între routerele de etaj și firewall |

Detaliile fiecărei subrețele, inclusiv masca, adresa de broadcast și intervalul de adrese utilizabile, sunt păstrate în planul de adresare tehnic al proiectului.

### 5.2. Switching și VLAN-uri

Switch-ul Entry asigură conectivitatea Ethernet între componentele principale ale infrastructurii. Rețeaua centrală de tranzit utilizează VLAN 1, iar configurația include și VLAN-urile 25 și 99 la nivelul unor porturi dedicate.

Pentru LAN-urile etajelor, modulele EtherSwitch asigură conectarea terminalelor în domeniile locale de switching. Separarea traficului între departamente se realizează prin subrețele IPv4 și interfețele de rutare ale routerelor de etaj.

### 5.3. Rutare

Interconectarea subrețelelor este realizată prin rutare statică IPv4. Fiecare router de etaj deține rute pentru celelalte rețele interne și o cale implicită către infrastructura perimetrală. Această strategie oferă trasee explicite și previzibile pentru dimensiunea actuală a rețelei.

Rețeaua ClientData este conectată direct la Floor3 și are un gateway propriu, diferit de cel al LAN-ului administrativ.

## 6. Securitate perimetrală și servicii

### 6.1. Firewall ASA

Cisco ASA 5506-X este amplasat între rețeaua internă de tranzit și routerul External. Acesta reprezintă punctul de control pentru comunicațiile dintre infrastructura organizației și segmentul extern de test.

Politicile de acces sunt organizate prin interfețele inside/outside și liste de control al accesului. În scenariul portalului web, firewall-ul permite publicarea controlată a serviciului HTTP prin asocierea adresei serverului intern cu o adresă de acces externă.

### 6.2. Network Address Translation

Mecanismul NAT este utilizat în scenariul de publicare a portalului web găzduit la Floor1. Serverul HTTP are adresa internă `192.168.10.10`, iar accesul din segmentul extern de test se realizează prin adresa de publicare `10.0.0.5`.

În acest flux, firewall-ul asociază adresa externă cu serverul intern și aplică politica de acces pentru traficul HTTP. Demonstrația oferă un exemplu de integrare a rutării, traducerii de adrese și filtrării la perimetru.

### 6.3. Servicii de aplicație

| Serviciu | Amplasare | Rol |
|---|---|---|
| HTTP | Floor1 | Portal web și demonstrația accesului prin NAT |
| FTP | Floor1 | Transfer de fișiere în infrastructura internă |
| Servicii de laborator | Floor2 | Teste controlate de conectivitate și securitate |
| Syslog | Floor3 | Centralizarea mesajelor emise de routere |
| ClientData | Segmentul redundant Floor3 | Resursă accesibilă stației PC0 |

## 7. Continuitatea serviciilor prin STP

Segmentul ClientData utilizează **Rapid Per-VLAN Spanning Tree (Rapid PVST+)** pentru gestionarea legăturilor redundante dintre Switch1, Switch2 și Switch3. Topologia include atât conexiuni paralele între switch-uri, cât și căi alternative prin triunghiul de interconectare.

Switch1 este root bridge pentru VLAN 1, iar Switch2 are prioritate de root secundar. Porturile aflate în starea forwarding transportă traficul curent, în timp ce porturile alternate sunt pregătite să participe la reconvergență.

Validarea implementării a inclus identificarea root bridge-ului, observarea rolurilor și stărilor porturilor, testarea comunicației dintre PC0 și ClientData și un exercițiu de întrerupere și restaurare a unui link activ. După restabilirea conexiunii, infrastructura a revenit la funcționarea normală.

**Valoarea pentru organizație:** accesul la resursa ClientData beneficiază de alternative de conectivitate la nivelul infrastructurii Ethernet, fără schimbări în configurarea stațiilor.

![[TestOSPF.png]]

*Figura 3. Test de redundanță în segmentul ClientData.*
## 8. Administrarea securizată prin SSH

Administrarea routerelor Floor1–Floor3 utilizează **Secure Shell, versiunea 2**, pentru sesiuni de management la distanță. Accesul se realizează prin autentificare locală, pe un canal criptat, iar liniile de acces configurate acceptă conexiuni SSH.

Sesiunile demonstrate pe Floor1 și Floor2 confirmă disponibilitatea mecanismului de administrare. Pentru Floor2 au fost observate explicit sesiunea activă, utilizatorul conectat, versiunea SSH și parametrii de protecție a comunicației.

**Valoarea pentru organizație:** echipa IT poate efectua verificări de stare și operațiuni de mentenanță asupra routerelor fără acces fizic la fiecare etaj.

![[SSH_Floor1_CorporatePC.png]]

*Figura 5. Acces administrativ SSH către Floor1.*

![[SSH_Floor2_Admin.png]]

*Figura 6. Sesiune SSH către Floor2.*

## 9. Monitorizare centralizată prin SNMP

Stația **SNMP-Monitor**, conectată la switch-ul Entry, asigură un punct central de interogare a echipamentelor de rețea. Aceasta utilizează MIB Browser pentru a consulta obiecte de management furnizate de agenții SNMP de pe routerele de etaj.

Implementarea utilizează **SNMPv2c** cu o comunitate de tip read-only. Prin interogări GET pot fi consultate numele dispozitivelor, timpul de funcționare și alte informații expuse de MIB-urile echipamentelor.

În testele realizate, interogarea `sysUpTime` a returnat timpul de funcționare pentru Floor1. Au fost consultate și informațiile de identificare ale routerelor Floor2 și Floor3.

**Valoarea pentru organizație:** parametrii echipamentelor pot fi consultați dintr-un singur punct, oferind personalului tehnic vizibilitate asupra infrastructurii de comunicații.

![[SNMP_Manager_Topology.png]]

*Figura 7. Poziționarea stației SNMP-Monitor în rețeaua de tranzit.*

![[SNMP_SysTime.png]]

*Figura 8. Interogarea SNMP a timpului de funcționare.*

![[GetName_Floor2_SNMP.png]]

*Figura 9. Identificarea routerului Floor2 prin SNMP.*

![[GetName_Floor3_SNMP.png]]

*Figura 10. Identificarea routerului Floor3 prin SNMP.*

## 10. Centralizarea evenimentelor prin Syslog

Serverul Syslog din rețeaua administrativă Floor3 colectează evenimente transmise de routerele celor trei etaje. Mesajele înregistrate includ schimbări de stare ale interfețelor și evenimente asociate administrării configurației.

Syslog completează monitorizarea SNMP și accesul SSH. Într-un flux de lucru operațional, mesajele colectate indică evenimentul, SNMP oferă informații despre dispozitiv, iar SSH permite investigarea configurației și intervenția administratorului.

| Etapă | Componentă | Funcție |
|---|---|---|
| Observarea evenimentului | Syslog | Centralizarea mesajelor de sistem |
| Consultarea stării | SNMP-Monitor | Interogarea obiectelor de management |
| Investigare | SSH | Acces la instrumentele de diagnostic ale routerului |
| Confirmarea restaurării | ICMP și monitorizare | Verificarea conectivității și a stării serviciului |

**Valoarea pentru organizație:** administrarea este susținută de o evidență centralizată a evenimentelor și de informații operaționale colectate de la echipamente.

## 11. Validarea infrastructurii

În cadrul implementării au fost efectuate teste de conectivitate, funcționare a protocoalelor și comportament al legăturilor redundante.

| Componentă verificată | Metodă | Rezultat evidențiat |
|---|---|---|
| Comunicație între routerele de etaj | Teste ICMP în rețeaua de tranzit | Răspunsuri de la routerele Floor1–Floor3 |
| Acces PC0 la gateway | ICMP către Floor3 | Conectivitate confirmată |
| Acces PC0 la ClientData | ICMP către server | Conectivitate confirmată |
| Management Switch1–Switch3 | ICMP către adresele de management | Răspunsuri de la toate cele trei switch-uri |
| STP | Observarea root bridge-ului și a rolurilor porturilor | Topologie cu forwarding și căi alternative |
| Redundanță STP | Întreruperea controlată a unei legături | Continuitatea comunicației raportată în testul practic |
| SSH | Sesiuni de administrare | Autentificare reușită pe Floor1 și Floor2 |
| SNMP | Interogări MIB Browser | Răspunsuri pentru uptime și identificarea dispozitivelor |
| Syslog | Observarea jurnalelor centralizate | Evenimente colectate de la toate cele trei etaje |
| NAT și acces HTTP | Acces prin adresa de publicare externă | Test HTTP consemnat în documentația proiectului |
## 12. Corespondența cu cerințele proiectului

### 12.1. Partea I: proiectarea rețelei

| Cerință | Element prezent în proiect | Secțiune |
|---|---|---|
| Alegerea și detalierea scenariului | Companie de servicii digitale, organizată pe trei etaje | 1 și 3 |
| Inventarul echipamentelor și al utilizatorilor | Routere, switch-uri, firewall, stații, servere și imprimante | 3 și 4 |
| Schema de adresare IPv4 | Subrețele pentru etaje, tranzit, perimetru și ClientData | 2 și 5 |
| Alegerea hardware și topologia logică | Rolurile echipamentelor și reprezentarea arhitecturii | 2 și 4 |
| VLAN-uri și switching | VLAN de tranzit și acces local prin module EtherSwitch | 5 |
| Rutarea rețelelor | Rutare statică IPv4 între etaje | 5 |
### 12.2. Partea a II-a: patru teme de implementare și analiză

| Temă     | Implementare în arhitectură                | Dovadă asociată                                                     |
| -------- | ------------------------------------------ | ------------------------------------------------------------------- |
| **STP**  | Rapid PVST+ pe Switch1–Switch3             | Root bridge, porturi forwarding/alternate și test de redundanță     |
| **SSH**  | Administrarea securizată a routerelor      | Sesiuni SSH și informații despre autentificare                      |
| **SNMP** | Monitorizarea routerelor prin SNMP-Monitor | Interogări GET, identificarea obiectelor MIB și valorile returnate  |
| **NAT**  | Publicarea portalului HTTP prin ASA        | Acces prin adresa NAT, politici de acces și verificări de traducere |
### 12.3. Elemente suplimentare

Pe lângă cele patru teme selectate, arhitectura include colectarea centralizată Syslog, filtrarea traficului prin ACL, distribuirea funcțională a serviciilor și un laborator dedicat evaluării securității. Aceste componente susțin scenariul de business și permit prezentarea unui flux complet de operare: furnizarea serviciului, monitorizarea infrastructurii, diagnosticarea unui eveniment și intervenția prin management securizat.

## 13. Organizarea documentației tehnice

Prezentul README descrie scenariul, arhitectura și modul în care elementele proiectului răspund cerințelor. Configurațiile de dispozitiv, ieșirile CLI și testele detaliate sunt organizate în documente individuale.

| Document | Conținut |
|---|---|
| `01_Network_Addressing.md` | Adresare, subrețele și inventar IP |
| `02_Routing_and_VLANs.md` | Switching, VLAN-uri, rutare statică și conexiuni |
| `03_Firewall_and_NAT.md` | Arhitectura perimetrală, politici și teste NAT |
| `04_STP.md` | Rapid PVST+, rolurile switch-urilor și testul de failover |
| `05_SSH.md` | Administrare securizată și analiza sesiunilor |
| `06_SNMP.md` | Agenți, MIB Browser, OID-uri și analiză trafic |
| `07_Syslog.md` | Colectarea și interpretarea jurnalelor |
| `08_Test_Results.md` | Matricea testelor și capturile de validare |

Fișierul Cisco Packet Tracer reprezintă implementarea executabilă a topologiei, iar capturile și documentele tehnice oferă suportul necesar pentru verificarea funcționalităților prezentate.

## 14. Concluzii

Proiectul Asterion Digital Services integrează într-o singură infrastructură funcțiile principale ale unei rețele enterprise: comunicație între departamente, furnizarea serviciilor, control perimetral, administrare securizată, monitorizare centralizată și redundanță locală.

Organizarea pe trei etaje și delimitarea rețelelor prin subrețele IPv4 oferă o structură clară pentru operare și administrare. Cisco ASA gestionează fluxurile la perimetru și publicarea portalului HTTP prin NAT. SSH susține administrarea de la distanță, SNMP permite consultarea parametrilor echipamentelor, iar Syslog centralizează evenimentele de sistem. Segmentul ClientData evidențiază utilizarea Rapid PVST+ pentru continuitatea conectivității la nivel Layer 2.

În ansamblu, soluția pune în relație cerințele tehnice ale proiectului cu operațiunile unei organizații care găzduiește servicii digitale, administrează resurse IT și desfășoară teste de securitate într-o infrastructură comună.

##### See also
[Detailed Configuration](./02_Configuration)

