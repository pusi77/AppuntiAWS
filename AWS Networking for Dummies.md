# AWS Networking for Dummies

# Table of contents

- [Table of contents](#table-of-contents)
- [VPC](#vpc)
  - [Regions e zone](#regions-e-zone)
  - [Subnet](#subnet)
    - [Sizing](#sizing)
    - [Routing](#routing)
      - [Quotas](#quotas)
    - [Indirizzi IP e hostname](#indirizzi-ip-e-hostname)
    - [Elastic IP](#elastic-ip)
      - [Quotas](#quotas-1)
    - [Network interfaces](#network-interfaces)
      - [Quotas](#quotas-2)
  - [Security](#security)
    - [Security groups](#security-groups)
    - [Network ACLs](#network-acls)
      - [Network ACLs vs Security Group](#network-acls-vs-security-group)
      - [Quotas](#quotas-3)
  - [VPC Peering](#vpc-peering)
    - [Costi](#costi)
    - [Limitazioni](#limitazioni)
      - [Quotas](#quotas-4)
  - [Quotas](#quotas-5)
- [NAT Gateway](#nat-gateway)
  - [Costi](#costi-1)
  - [NAT instances](#nat-instances)
  - [Quotas](#quotas-6)
- [Route53](#route53)
  - [Formato dei domain name](#formato-dei-domain-name)
  - [Pricing](#pricing)
  - [Hosted zones](#hosted-zones)
    - [Split-view DNS](#split-view-dns)
  - [Routing policies](#routing-policies)
  - [Alias record](#alias-record)
  - [Record supportati](#record-supportati)
  - [DNS Firewall](#dns-firewall)
- [Direct Connect](#direct-connect)
  - [Hosting e infrastruttura](#hosting-e-infrastruttura)
  - [Virtual interface](#virtual-interface)
  - [Costi](#costi-2)
- [Site-to-Site VPN](#site-to-site-vpn)
  - [Virtual Private Gateway](#virtual-private-gateway)
  - [Transit Gateway](#transit-gateway)
    - [Routing](#routing-1)
  - [Customer Gateway](#customer-gateway)
    - [Customer Gateway Device](#customer-gateway-device)
  - [Autenticazione](#autenticazione)
  - [Architetture](#architetture)
- [CloudFront](#cloudfront)
  - [Casi d'uso](#casi-duso)

# VPC

// sources: [Overview of VPCs and subnets](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Subnets.html)
Network virtuale logicamente isolato all'interno di AWS. In ogni VPC deve essere specificato uno spazio di indirizzamento IPv4 sotto forma di CIDR block ([RFC4632](https://tools.ietf.org/html/rfc4632)). Una VPC si estende su tutte le Availabilkity Zone di una Region.

## Regions e zone

// source: [Regions and Zones](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html)
I servizi AWS sono hostati in più tipi di zone:

- **Regions**: progettate per essere isolate le une dalle altre, per questo motivo AWS non replica automaticamente le risorse tra le regions (ad esempio due AMI uguali hanno ID diversi tra le region).
- **Availability Zones** (AZs): ulteriori zone isolate contenute nelle region, utilizzate per garantire alta disponibiità tramite la replicazione delle risorse. Le AZs sono identificate da ID composti dal codice della region seguito da una lettera (esempio: `use-east-1a`), per distribuire meglio le risorse l'associazione tra la lettera finale e l'AZ cambia tra gli account. AWS si riserva il diritto di limitare il lancio di istanze in un nuova AZ se questa ha risorse limitate.
- **Local Zones**: estensione di una region in prossimita degli utenti finali, possiedono connessione ad internet e supportano il Direct Connect. Usate per minimizzare la latenza per l'utente. Sono identificate da ID composti dal codice della region seguito da un identificativo (esempio: `us-west-2-lax-1a`). Supportano solo un limitato insieme di servizi AWS: EC2, EBS, ECS, EKS, IG, ALB.
- **Wavelenght Zones**: zone isolate situate presso i network degli operatori telefonici, utilizzano la rete 5G fornita da questi per minimizzare la latenza verso dispositivi mobili e utenti finali. Le wavelenght zones estendono logicamente le VPC. Sono identificate da ID composti dal codice della region seguito da un identificativo (esempio: `us-east-1-wl1-bos-wlz-1`).
- **AWS Outposts**: servizio fully managed che prevede il deploy di infrastruttura AWS presso il cliente, utilizzato per garantire latenze minime e possesso dei dati.

## Subnet

// sources: [Overview of VPCs and subnets](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Subnets.html)
Range di indirizzi IP all'interno della VPC. Ogni Subnet è definita da un CIDR block. Ogni Subnet si trova all'interno di una sola Availability Zone (utile per limitare l'impatto del down di una AZ).
Le subnet possono essere di più tipi:

- IPv4-only: le comunicazioni tra i servizi interni alla subnet possono avvenire solo su IPv4.
- Dual-stack (IPv4 and IPv6): le comunicazioni tra i servizi interni alla subnet possono avvenire su IPv4 o IPv6.
- IPv6-only: le comunicazioni tra i servizi interni alla subnet possono avvenire solo su IPv6.

Le subnet possono essere suddivise in più classi:

- Public: subnet avente rotta 0.0.0.0/0 diretta ad un Internet Gateway.
- Private: subnet non avente rotte dirette all'esterno.
- Natted: subnet avente rotta 0.0.0.0/0 diretta ad un Nat Gateway.

### Sizing

Sia per le VPC che per le subnet, la dimensione della maschera di sottorete deve essere compresa tra **/16**(65,536 IPs) e **/28**(16 IPs). Per le subnet *si raccomanda* di attenersi agli spazi di indirizzamento definiti dall'[RFC1918](http://www.faqs.org/rfcs/rfc1918.html), ovvero:

- 10.0.0.0/8 -> 10.0.0.0/16 su AWS
- 172.16.0.0/12 -> 172.16.0.0/16 su AWS
- 192.168.0.0/20

In ogni subnet 5 indirizzi IP sono riservati ad AWS:

- x.x.x.0: indirizzo della rete.
- x.x.x.1: riservato per il VPC router.
- x.x.x.2: riservato per il server DNS.
- x.x.x.3: riservato per usi futuri.
- x.x.x.255: indirizzo broadcast, riservato nonostante il broadcast non sia supportato nelle VPC.

In seguito alla creazione le dimensioni delle subnet non possono più essere modificate.

### Routing

// sources: [Managing route tables for your VPC](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html)
Ogni subnet deve essere associata ad una **Route Table** che specifica le rotte per il traffico in uscita dalla subnet. A ogni VPC viene associata una Main Route Table alla creazione. Le rotte sono espresse tramite una **Destination**, espressa come blocco CIDR, e un **Target**, che corrisponde a un gateway/interface/ecc.

#### Quotas

|                                      | Default | Soft-limit | Commenti                                                                                |
| ------------------------------------ |:-------:|:----------:| --------------------------------------------------------------------------------------- |
| Route tables per VPC                 | 200     | Sì         |                                                                                         |
| Rotte per route table                | 500     | Sì         | Massimo 1000, ma le network performance ne risentono. IPv4 e IPv6 hanno totali sepatati |
| Rotte BGP advertised per route table | 100     | No         |                                                                                         |

### Indirizzi IP e hostname

Quando un'istnaza viene lanciata all'interno di una subnet, all'istanza viene assegnata un scheda di rete virtuale a cui sono associati un IP privato e un hostname interno. Di default le VPC operano con IPv4, ma è possibile impostare la VPC in modo che operi in modalità **dual-stack**, ovvero tramite indirizzi IPv4, IPv6 o entrambi. L'hostname assegnato all'istanza può essere di due tipi: resource based o IP-based.
È anche prevista la possibilità di utilizzare i propri range IP importandoli su AWS (BYOIP), il blocco più specifico importabile è /24 e /48 per IPv6.

### Elastic IP

// source: [Elastic IP addresses](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html)
Sono indirizzi IPv4 statici che possono essere associati a un'istanza di un servizio, possono essere utilizzati per reagire a un guasto rimappandoli su un'altra istanza del servizio o per l'associazione al record DNS del proprio dominio. Gli Elastic IP, previa richiesta, sono utilizzabili gratuitamente, ma se lasciati inutilizzati è necessario pagare una tariffa oraria. 

#### Quotas

|                       | Default | Soft-limit |
| --------------------- |:-------:|:----------:|
| Elastic IP per region | 5       | Sì         |

### Network interfaces

// source: [Elastic network interfaces](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html)
Un'elastic network inteface è un componente logico che rappresenta una scheda di rete virtuale all'interno di una VPC. Possiede un indirizzo IPv4 privato primario e un indirizzo MAC e può possedere uno o più secondari, un Elastic IP, un indirizzo IPv4 pubblico, uno o più indirizzi IPv6, security group. Possono essere attacacte e staccate dalle istanze a piacere, mantenendo gli stessi attributi e redirigendo il traffico alla nuova istanza.

#### Quotas

|                                | Default                                                                                                      | Soft-limit |
| ------------------------------ |:------------------------------------------------------------------------------------------------------------:|:----------:|
| Network interface per instance | [Dipende dall'istanza](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI) | No         |
| Network interface per region   | 5000                                                                                                         | Sì         |

## Security

A livello di rete sono disponibili diversi strumenti per il controllo del traffico:

### Security groups

// source: [Control traffic to resources using security groups](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html)
Sono firewall distribuiti, associati a un'istanza, ne controllano il traffico in ingresso e uscita. Sui security group possono essere specificate solo *allow rules* e non *deny rules*. I security group di default non hanno regole in ingresso, di conseguenza non è permesso traffico diretto verso l'istanza, ma essendo **stateful** permettono sempre il traffico di ritorno da una richiesta in uscita. Il traffico in uscita di default è sempre permesso, per limitarlo è necessario rimuovere la regola e impostare regole più ristrette. Ad un'istanza possono essere associati più security group, le regole imposte ad essa saranno quindi date dalla loro unione.
Le rule sono composte da:

- **Protocol**
- **Port range**
- ICMP type and code
- **Source or destination**: sorgente o destinazione a seconda del tipo di regola, le risorse specificate possono essere indirizzi IPv4 (e IPv6) sia singoli che range, ID di servizi.

|                          | Default | Soft-limit | Commenti                                                                        |
| ------------------------ |:-------:|:----------:| ------------------------------------------------------------------------------- |
| SG per region            | 2500    | Sì         |                                                                                 |
| Rules per SG             | 60      | Sì         | Inbound e outbound, IPv4 e IPv6 rules hanno totali separati                     |
| SG per network interface | 5       | Sì         | Questa quota moltiplicata per quella delle rules per SG non può superare i 1000 |

### Network ACLs

// source: [Control traffic to subnets with Network ACLs](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html)
Utilizzate per controllare il traffico in ingresso e uscita da una subnet. Di default le custom NACLs impediscono tutto il traffico IPv4, e nel caso anche IPv6, in entrambe le direzioni, mentre la NACL di default ne permette il passaggio. Sono composte da una lista numerata di regole, vengono valutate in ordine crescente fino a un massimo di 32766 regole. Le NACLs hanno regole separate per ingresso e uscita, e ognuna di esse può essere di tipo *allow* o *deny* Ogni subnet può essere associata ad una sola NACL per volta.
Le regole sono composte da:

- **Rule number**
- **Type**: tipo di traffico (esempio: SSH).
- **Protocol**
- **Port range**
- **Source or destination**: sorgente o destinazione a seconda del tipo di regola, le risorse specificate possono essere solo indirizzi IPv4 (e IPv6) sia singoli che range.
- **Allow/Deny**

#### Network ACLs vs Security Group

// source:[Internetwork traffic privacy in Amazon VPC](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Security.html#VPC_Security_Comparison)

| Security group                                       | Network ACL                                                         |
| ---------------------------------------------------- | ------------------------------------------------------------------- |
| Instance level                                       | Subnet level                                                        |
| Solo *allow rules*                                   | Sia *allow* che *deny rules*                                        |
| Stateful                                             | Stateless                                                           |
| Valuta tutte le regole prima di applicarle           | Valuta le regole in ordine crescente                                |
| Applicato solo alle istanze esplicitamente associate | Applicato automaticamente a tutte le istanze presenti in una subnet |

![Example](https://docs.aws.amazon.com/vpc/latest/userguide/images/security-diagram.png)

#### Quotas

|                | Default | Soft-limit | Commenti                                       |
| -------------- |:-------:|:----------:| ---------------------------------------------- |
| NACLs per VPC  | 200     | Sì         | Le NACLs possono essere associate a più subnet |
| Rules per NACL | 20      | Sì         | IPv4 e IPv6 hanno totali separati              |

## VPC Peering

// source: [What is VPC peering?](https://docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.html)
// esempi: [Configurations with routes to an entire CIDR block](https://docs.aws.amazon.com/vpc/latest/peering/peering-configurations-full-access.html), [VPC peering configurations](https://docs.aws.amazon.com/vpc/latest/peering/peering-configurations-full-access.html)
Permette l'interconnessione di due VPC, consentendo lo scambio di traffico IPv4 e IPv6 tra di esse come se fossero nella stessa rete. Per questa operazione AWS non utilizza gateway strani o VPN, è solo un'operazione logica, che non richiede hardware. Il peering può essere effettuato anche tra region differenti, in tal caso il traffico è criptato e non subisce bottleneck sulla banda in quanto sempre sulla backbone AWS, non passa quindi sull'internet pubblico.
Sebbene il peering sia una relazione uno a uno tra le VPC, è possibile comunque fare un peering tra una VPC già facente parte di un altro e una terza VPC. Non sono però consentite relazioni transitive tra i peering.

### Costi

// source: [What is VPC peering?](https://docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.html)
Il peering tra VPC è gratuito, ciò che costa è il traffico di dati sul peering: all'interno della stessa region è $0.01/GB in entrambe le direzioni.

### Limitazioni

// source: [VPC peering basics](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-basics.html)

- Non è possibile creare un peering in caso di overlapping tra i CIDR delle VPC.
- Il peering non è transitivo.
- IPv6 è supportato, ma non automaticamente.
  Se il peering è inter-region ci sono limitazioni addizionali:
- Non è possibile creare security group rule che referenzino un security group in una peer VPC.
- È necessario abilitare il supporto alla risoluzione DNS sul peering per poter risolvere gli hotname DNS privati per la 
- peered VPC. 

#### Quotas

|                            | Default | Soft-limit | Commenti                                                                                         |
| -------------------------- |:-------:|:----------:| ------------------------------------------------------------------------------------------------ |
| VPC peering attivi per VPC | 50      | Sì         | Massimo 125. Se viene aumentato è necessario aumentare anche il numero di rotte per route table. |

## Quotas

|                      | Default | Soft-limit | Commenti    |
| -------------------- |:-------:|:----------:| ----------- |
| VPCs per region      | 5       | Sì         | Massimo 100 |
| Subnets per VPC      | 200     | Sì         |             |
| Blocchi IPv4 per VPC | 5       | Sì         | Massimo 50  |
| Blocchi IPv6 per VPC | 1       | No         |             |

# NAT Gateway

// source: [NAT gateways](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html), [NAT gateway basics](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)
// esempi: [NAT gateway use cases](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-scenarios.html)
Servizio di NAT gestito, permette alle istanze in una VPC privata di connettersi a risorse esterne, ma non permette a risorse esterne di iniziare una connessione alle istanze. Può essere anche utilizzato per connettersi ad altre VPC o a reti on-premises, in tal caso il traffico in uscita va diretto attraverso un **transit gateway** o un **virtual private gateway**. Il traffico in uscita non può invece essere diretto a VPC peering, Site-to-Site VPN o Direct Connect e vice versa.
Alla creazione è presente un flag Public/Private che permette di disabilitare la connessione ad internet.
È importante ricordare che il NAT sostituisce l'indirizzo IP sorgente da quello dell'istanza a quello del NAT gateway, in particolare se Public con il suo Elastic IP. Inoltre al NAT gateway non possono essere associati security group, di conseguenza vanno associati alle singole istanze, possono essere invece utilizzate network ACL (a livello di subnet).

| Specs       |                                  |
| ----------- | -------------------------------- |
| Protocols   | TCP, UDP, ICMP                   |
| IP version  | IPv4, IPv6 (NAT64+DNS64)         |
| Bandwidth   | 5Gbps (scala fino a 45Gbps)      |
| Packets/sec | 1M (scala fino a 4M, poi droppa) |
| Connections | 55K (~900/sec per *destination*) |

## Costi

Il costo del NAT gateway è calcolato sia in base a quante ore è acceso, sia a quanti GB di dati processa (il prezzo di entrambi è circa $0.048, quindi circa 30€ al mese, dati esclusi).

## NAT instances

Le NAT AMI sono basate su una versione di Amazon Linux che ha raggiunto la EOL a fine 2020, di conseguenza AWS raccomanda la migrazione a NAT gateway. Inoltre l'utilizzo di  una NAT instance non supporta il traffico IPv6.

## Quotas

|                     | Default | Soft-limit |
| ------------------- |:-------:|:----------:|
| NAT Gateways per AZ | 5       | Sì         |

# Route53

// sources: [What is Amazon Route 53?](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html), [Amazon Route 53 concepts](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/route-53-concepts.html)
Servizio DNS gestito, può essere utilizzato per 3 scopi principali:

- **Registrazione domini**: alla creazione di un nuovo dominio viene automaticamente creata una hosted zone omonima. Il dominio viene registrato tramite Amazon Registrar o tramite il registrar associato, Gandi. 

- **Routing DNS**

- **Health checking**: dovrà essere specificato l'IP/ domain name, il protocollo (HTTP/S o TCP), la **failure threshold** (quanti fallimenti consecutivi rendono l'istanza unhealthy) e opzionalmente il metodo di notifica del problema. Se viene configurata la notifica, R53 setta di default un allarme CloudWatch, che utilizzerà poi un topic SNS per notificare l'utente.

## Formato dei domain name

//sources: [DNS domain name format](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/DomainNameFormat.html)

I nomi di dominio (compresi quindi hosted zones e records) sono composti da una serie di labels separati da punti. ogni label può essere lungo al massimo 63 bytes e la lunghezza totale non può superare i 255 bytes, punti compresi. Nel nome di dominio possono essere inserite lettere, numeri e trattini (hyphen), ma questi ultimi non all'inizio o alla fine del label.

## Pricing

//source: [Amazon Route 53 pricing]([Amazon Route 53 pricing - Amazon Web Services](https://aws.amazon.com/route53/pricing/)
Per quanto riguarda il prezzo dell'acquisto del dominio si passa da una decina a centinaia di dollari a seconda del TLD.

## Hosted zones

//source: [Working with hosted zones](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zones-working-with.html)

Una hosted zone è un insieme di record, i quali contengono informazioni riguardo il routing del traffico verso un dominio e i suoi sottodomini. La hostedzone ha lo stesso nome del dominio al quale si riferisce. Sono di due tipi

- **Pubbliche**: contengono record esposti a internet.

- **Private**: contengono record raggiungibili all'interno delle VPC.

### Split-view DNS

//source: [Considerations when working with a private hosted zone](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zone-private-considerations.html)

Tramite Route 53 è possibile configurare lo split-view DNS, tramite cui viene utilizzato lo stesso nome di dominio sia su hosted zone pubblica che privata, seguendo quindi due rotte diverse.

## Routing policies

//source: [Choosing a routing plicy](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html)

Alla creazione di un record è richiesta la scelta di una routing policyche determina come Route 53 risponde alle richieste. Ci sono diversi tipi di routing policy:

- **SImple**: dirige il traffico a una singola risorsa.

- **Failover**: specifica una seconda risorsa a cui dirigere il traffico se la prima è *unhealty*

- **Geolocation**: dirige il traffico a una risorsa a seconda della zona geografica di provenienza della richiesta.

- **Latency**: dirige il traffico alla risorsa con RTT minore

- **Multivalue answer**: risponde alla richiesta DNS con fino a 8 IP *healty*.

- **Weighted**: dirige il traffico a più risorse a seconda della distribuzione selezionata tramite pesi.

## Alias record

//source: [Choosing between alias and non-alias records](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-choosing-alias-non-alias.html)

Route 53 offre un'estensione ai comuni record DNS detta **alias record**, ovvero un record che permette di dirigere il traffico verso una risorsa AWS, anche tra diverse hosted zones. Differisce da un record CNAME in quanto è permesso creare un alias anche per una *apex zone*. Inoltre l'utilizzo di record alias è gratuito.

## Record supportati

//source: [Supported DNS record types](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/ResourceRecordTypes.html)

I record supportati sono: 

- A (Host address): contiene un indirizzo IPv4.

- AAAA (IPv6 host address): contiene un indirizzo IPv6.

- CAA (Certification Authority Authorization): specifica quali CAs sono autorizzati a creare certificati per un dominio o sottodominio.

- CNAME (Canonical name for an alias): associa a un dominio un altro dominio o sottodominio.

- DS (Delegation Signer): utilizzati per DNSSEC.

- MX (Mail eXchange): utilizzato per i server mail.

- NAPTR (Naming Authority Pointer): utilizzato da DDDS.

- NS (Name Server): identifica in server DNS autoritativo per una zona.

- PTR (Pointer): mappa un IP a un nome di dominio corrispondente.

- SOA (Start Of Authority): contiene informazioni riguardo un dominio.

- SRV (Location of service): contiene informazioni riguardo priorità, peso, porta e un dominio.

- TXT (Descriptive text): contiene una o più stringhe comprese in doppi apici.

## DNS Firewall

//source: [How Route 53 Resolver DNS Firewall works](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-dns-firewall-overview.html)

Route 53 offre la possibilità di impostare un firewall a livello DNS che, tramite delle **DNS Firewall rule**, filtra le richieste di risoluzione di determinati domini specificati nelle **domain list**. Questa funzionalità offre solo il filtro sul nome di dominio, non può bloccare l'IP associato.

# Direct Connect

//source: [What is AWS Direct Connect?](https://docs.aws.amazon.com/directconnect/latest/UserGuide/Welcome.html)

Servizio che permette di creare interfacce virtuali on-premises per collegarsi direttamente a una VPC, o ad altri servizi pubblici, bypassando l'ISP. Per collegarsi direattamente a una VPC

## Hosting e infrastruttura

//source: [Getting Started with AWS Direct Connect](https://www.youtube.com/watch?v=y4rIwSbdlS0)

Ci sono 3 possibilità per quanto riguarda l'hosting e la gestione in generale del Direct connect:

- **Collocation**: è necessario accordarsi con dei AWS Direct Connect Partners presso cui portare la connessione fisica dal data center on-premises e l'infrastruttura necessaria al collegamento con AWS. Tutta l'infrastruttura è responsabilità dell'utente.

- **Direct Connect Partner**: in questo caso l'infrastruttura necessaria al collegamento con AWS è fornita e manutenuta dal partner, generalmente è richiesta solo la posa del cavo.

- **Direct Connect node**: è necessario accordarsi con AWS per raggiungere direttamente un nodo di Direct Connect. Tutta l'infrastruttura è responsabilità dell'utente.

Le larghezze di banda possibili sono 1/10/100Gbps, sono supportati sia IPv4 che IPv6.

## Virtual interface

//source: [Getting Started with AWS Direct Connect](https://www.youtube.com/watch?v=y4rIwSbdlS0)

Ci sono 3 tipi di interfacce virtuali per il collegamento ad AWS:

- **Private**: consente la connessione a tutte le VPC presenti nello spazio di indirizzamento privato. È possibile collegare una Private Virtual Interface a più VPC attraverso i **Private Gateways**, associandoli a un **Direct Connect Gateway**

- **Public**: consente la connessione a tutte le VPC presenti nello spazio di indirizzamento pubblico o alle risorse connesse ad un **Public Endpoint**

- **Transit**: permette la connessione tra il Direct Connect Gateway e fino a 3 **Transit Gateway**, che potranno poi essere connessi  a più VPC nella stessa regione (anche se facenti parte di account differenti).

## Costi

//source: [AWS Direct Connect pricing](https://aws.amazon.com/directconnect/pricing/)

Il costo di un Direct Connect dipende dal tipo di hosting scelto, dalla region in cui viene installato, dalla larghezza di banda scelta, dalla quantità di traffico in uscita e dal numero di ore di attività.

# Site-to-Site VPN

//source: [How AWS Site-to-Site VPN works](https://docs.aws.amazon.com/vpn/latest/s2svpn/how_it_works.html)

Una VPN Site-to-Site permette di creare un tunnel tra un *virtual private gateway* o un *transit gateway* su AWS e un *customer gateway* on-premises.

## Virtual Private Gateway

//source: [How AWS Site-to-Site VPN works](https://docs.aws.amazon.com/vpn/latest/s2svpn/how_it_works.html)

È la componente che concentra la VPN lato AWS.

![Virtual Private Gateway](https://docs.aws.amazon.com/vpn/latest/s2svpn/images/vpn-how-it-works-vgw.png)

## Transit Gateway

// source: [What is a transit gateway?](https://docs.aws.amazon.com/vpc/latest/tgw/what-is-transit-gateway.html)

Componente di più alto livello utilizzata per connettere VPCs e network on-premises. Una parte fondametale sono gli **attachments**, che permettono il collegamento di una o più VPCs, Direct Connect gateway, connessioni VPN e altri transit gateway in peering.

![Transit Gateway](https://docs.aws.amazon.com/vpn/latest/s2svpn/images/vpn-how-it-works-tgw.png)

### Routing

// source: [How transit gateways work](https://docs.aws.amazon.com/vpc/latest/tgw/how-transit-gateways-work.html) Il transit gateway possiede una *default route table*, che di base rappresenta la *default association route table* e la *default propagation rout table*. È permessa la creazione di route table aggiuntive in modo da isolare le subnet degli attachments, ogni attachment è associato ad una (ed una sola) route table. Gli attachments possono propagare le proprie rotte a una o più route table. Se sono presenti rotte uguali con indirizzo IP, ma target differenti, le rotte statiche hanno la precedenza su quelle propagate.

## Customer Gateway

//source: [How AWS Site-to-Site VPN works](https://docs.aws.amazon.com/vpn/latest/s2svpn/how_it_works.html)

Risorsa che rappresenta il *customer gateway device* sull'infrastruttura AWS.

### Customer Gateway Device

//source: [How AWS Site-to-Site VPN works](https://docs.aws.amazon.com/vpn/latest/s2svpn/how_it_works.html)

Si tratta di un dispositivo fisico o software installato on-premises.

## Autenticazione

//source: [Site-to-Site VPN tunnel authentication options](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-tunnel-authentication-options.html)

Esistono due opzioni principali per l'autenticazione:

- **Pre-shared keys**: opzione di default, consiste in una stringa immessa durante la configurazione del customer gateway device.

- **Certificato privato**: in alternativa è possibile utilizzare un certificato privato ottenuto da AWS Certificate Manager (firmato da ACM o da una CA esterna).

## Architetture

//source: [Site-to-Site VPN single and multiple connection examples](https://docs.aws.amazon.com/vpn/latest/s2svpn/Examples.html)

Non vale nemmeno la pena riportarle.

# CloudFront

//source: [What is Amazon CloudFront?](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html)

CloudFront è un servizio che svolge la funzione di caching per i contenuti statici e dinamici richiesti dagli utenti. CloudFront distribuisce i contenuti tramite una rete di data center detti *edge locations*, le richieste vengono indirizzate alla edge location che offre una minore latenza. Quando un utente richiede un contenuto, se esso è presente in una edge location, CloudFront lo restituisce immediatamente. Se non lo è, CloudFront richiama la *origin* definita come sorgente del contenuto.

## Casi d'uso

//soruce: [CloudFront use cases](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/IntroductionUseCases.html)

- **Cache per siti statici**: utilizzando un bucket S3 per hostare il sito e CloudFront impostato con una Origin Access Identity (OAI) per servirlo e limitare l'accesso solo tramite CloudFront.
- **Video on demand o in streaming live (VOD)**
- **Crittografia di specifici dati**: aggiungendo una chiave pubblica su CloudFront è possibile specificare alcuni dati da crittografare con la suddetta.
- **Personalizzazione di contenuti**: tramite il servizio Lambda@Edge è possibile creare funzioni che modificano i dati restituiti da CloudFront a seconda del richiedente o della modalità di richiesta.
- **Contenuti privati**: utilizzando le Lambda@Edge è possibile servire contenuti privati richiedendo l'accesso al contenuto tramite singed URLs o signed cookies.

# Elastic Load Balancing

//source: [How Elastic Load Balancing works](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html)

L'Elastic Load Balancing permette di distribuire automaticamente il traffico in arrivo su più risorse: istanze EC2, container, indirizzi IP, anche su più Availability Zone. L'ELB monitora lo stato dei delle risorse associate e dirige il traffico a quelle healthy.

Alla creazione di un load balancer viene richiesto in quali AZ abilitarlo, una sua replica verrà creata all'interno di ogni AZ selezionata. È inoltre possibile abilitare **cross-zone load balancing** in modo che ogni load balancer possa distribuire traffico su ogni AZ anziché solo in quella in cui si trova. Questo è utile al fine di una distribuzione più uniforme del traffico in arrivo.

<img title="" src="file:///C:/Users/Andrea/AppData/Roaming/marktext/images/2022-04-04-11-59-55-component_architecture.png" alt="" data-align="left">

## Listener

//source: [Listeners for your Application Load Balancers](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html)

Il load balancer va configurato in modo che accetti traffico in ingresso tramite uno o più listeners. Un listener è un processo che rimane in attesa di richieste di connessione sa parte del client tramite uno specifico protocollo e porta. I listener possiedono delle **listener rule** che consistono di:

- *Priority*: le listener rule vengono valutate in base alla priorità, partendo dai valori inferiori e andando in ordine crescente fino alla default rule, valutata per ultima.

- *Actions*: ogni action ha un tipo, un ordine e le informazioni necessarie ad eseguirla.

- *Conditions*: ogni condizione ha un tipo e delle informazioni di configurazione. Quando la condizione di una rule è soddisfatta, le sue azioni vengono eseguite.

## Target Group

//source: [Target groups for your Application Load Balancers](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html)

Le risorse associate ad un ELB sono registrate in target group, all'interno di cui vengono eseguiti degli health check per stabilire quali delle risorse presenti sono healty. Ogni risorsa può essere registrata in più target group.

Alla creazione di un target group è richiesto un **target type** che identifica il tipo di risorsa target, i possibili tipi sono *instance*, *ip* o *lambda*.

## Comparison

//source: [Elastic Load Balancing features](https://aws.amazon.com/elasticloadbalancing/features/)

| Feature     | ALB                  | NLB               | GLB          | CLB |
| ----------- |:--------------------:|:-----------------:|:------------:|:---:|
| Livello OSI | 7 (Application)      | 4 (Transport)     | 3 (Network)  | 4/7 |
| Target      | IP, Instance, Lambda | IP, Instance, ALB | IP, Instance |     |
| Protocolli  | HTTP, HTTPS, gRPC    | TCP, UDP, TLS     | IP           |     |
| Slow start  | Sì                   | No                | No           | No  |