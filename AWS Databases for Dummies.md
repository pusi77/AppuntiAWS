# RDS

//source: [What is Amazon Relational Database Service (Amazon RDS)?](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Welcome.html)

Amazon Relational Databases Service (RDS) è il servizio che permette di configurare ed utilizzare database relazionali 

su AWS. in particolare RDS supporta i seguenti *engine*:

- MySQL

- MariaDB

- PostgreSQL

- Oracle

- Microsoft SQL Server

## DB instance

//source: [Amazon RDS DB instances](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Overview.DBInstance.html)

Con DB instance si intende un'istanza isolata che può contenere più database creati dall'utente ed è accessibile tramite i tool classici.

### Limiti

//source: [Amazon RDS DB instances](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Overview.DBInstance.html)

Per ogni account è possibile avere al massimo 40 RDS DB instances (Soft Limit), rispettando inoltre le seguenti limitazioni:

- 10 per ogni edizione SQL Server utilizzando il modello "license-includes"

- 10 per Oracle utilizzando il modello "license-included"

## DB instance class

//source: [DB instance classes](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.DBInstanceClass.html)

L'instance class determina la capacità computazionale e di memoria di un'istanza RDS ed è composta da un tipo di istanza e una dimensione.

I nomi delle classi sono espressi con una sintassi simile a quelli delle EC2: `db.[classe][generazione][caratteristiche]`
Sono presenti tre tipi di instance class:

- **standard - "m"**: general-purpose, CPU, RAM e Networking sono bilanciati

- **memory optimized - "x/z/r"**: ottimizzate per worload sbilanciati sull'utilizzo della memoria.

- **burstable performance - "t"**: utilizzano un sistema di crediti che consente di utilizzare l'intera CPU per brevi finestre temporali.