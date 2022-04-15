# AWS Compute for Dummies

- [AWS Compute for Dummies](#aws-compute-for-dummies)
- [ECS](#ecs)
  - [Cluster](#cluster)
  - [Task definition](#task-definition)
  - [Task](#task)
  - [Service](#service)
    - [Deployment type](#deployment-type)
      - [Rolling update](#rolling-update)
      - [Blue/Green deployment](#bluegreen-deployment)
      - [External deployment](#external-deployment)
    - [Load Balancing](#load-balancing)
  - [Quotas](#quotas)
# ECS

//source: [What is Amazon Elastic Container Service?](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html)

Amazon Elastic Container Service (ECS) è un servizio che permette di gestire container Docker. Su ECS i container sono definiti tramite una **task definition**, utilizzata per lanciare singoli **task** o task facenti parte di un **service**, ovvero una configurazione che permette di gestire simultaneamente task all'interno di un **cluster**. 
I task e i service possono essere lanciati su due tipi di infrastruttura:

- **Fargate**: ovvero un'infrastruttura serverless gestita da AWS

- **EC2**: oppure sulle classiche EC2, per avere più controllo.

## Cluster

//sources: [Amazon ECS components](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/welcome-features.html), [Amazon ECS clusters](https://docs.aws.amazon.com/AmazonECS/latest/userguide/clusters.html)

Un cluster ECS è un gruppo logico di task o service utilizzato per isolare più applicazioni. Attraverso l'uso di cluster distinti è possibile essere certi che le diverse applicazoni non utilizzino la stessa infrastruttura sottostante. La capacità infrastrutturale può essere fornita al cluster mediante Fargate, EC2, server on-premises e macchine virtuali.

## Task definition

//sources: [Amazon ECS components](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/welcome-features.html), [Amazon ECS task definitions](https://docs.aws.amazon.com/AmazonECS/latest/userguide/task_definitions.html)

Si tratta di un file JSON che descrive uno o più container che costituiscono l'applicazione. All'interno specifica vari parametri dell'applicazone, tra cui: le immagini Docker da usare per ogni container all'interno del task, le quantità di CPU e RAM pero ogni task, il ruolo IAM da assegnare al task, eventuali volumi da montare ecc.
Un'applicazione può usare più task definition.

Listone dei parametri della task definition [qui](https://docs.aws.amazon.com/AmazonECS/latest/userguide/task_definition_parameters.html).

Esempi di task definition [qui](https://docs.aws.amazon.com/AmazonECS/latest/userguide/example_task_definitions.html).

## Task

//source: [Amazon ECS components](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/welcome-features.html)

Un task è un'istanza della task definition all'interno di un cluster.

## Service

//source: [Amazon ECS services](https://docs.aws.amazon.com/AmazonECS/latest/userguide/ecs_services.html)

Tramite l'utilizzo di un service è possibile mantenere il numero desiderato di task in esecuzione. Se uno di questi si ferma, viene prontamente sostituito in modo da mantenere il numero di task desiderati.

Listone dei parametri del service [qui](https://docs.aws.amazon.com/AmazonECS/latest/userguide/service_definition_parameters.html).

### Deployment type

// source: [Amazon ECS Deployment types](https://docs.aws.amazon.com/AmazonECS/latest/userguide/deployment-types.html)

Ci sono tre tipi di rilascio supportati da ECS:

#### Rolling update

//source: [Rolling update](https://docs.aws.amazon.com/AmazonECS/latest/userguide/scheduling_tasks.html)

Quando viene avviato il deployment, lo scheduler di ECS sostituisce i task attivi con dei nuovi task. Il numero di task che ECS aggiunge o rimuove è determinato dai parametri del service

- *minimumHealtyPercent*: percentuale minima di task attivi durante un deployment rispetto al desired count, approssimata per eccesso.

- *maximumPercent*: percentuale massima di task attivi durante un deployment rispetto al desired count, approssimata per difetto.

È importare settare questi parametri in modo che durante un deploy sia possibile aggiungere o terminare almeno un task.

#### Blue/Green deployment

//source: [Blue/Green deployment with CodeDeploy](https://docs.aws.amazon.com/AmazonECS/latest/userguide/scheduling_tasks.html)

È un tipo di rilascio controllato da CodeDeploy mediante cui è possibile cerificare il nuovo deployment prima di dirottarvi il traffico di produzione.

Ci sono tre modi in cui il traffico può essere dirottato durante un blue/green deployment:

- **Canary**: inizialmente solo una percentuale del traffico viene dirottata ai task aggiornati, la restante parte viene dirottata con un certo ritardo.

- **Linear**: una quantità costante di traffico è dirottata sui task aggiornati ad intervalli regolari.

- **All-at-once**: il traffico è dirottato ai task aggiornati tutto in una volta.

Se il durante il deployment avvengono eventi dovuti all'autoscaling del servizio, è possibile che si verifichino rallentamenti, o in certi casi il fallimento, del deploy. 

#### External deployment

//source: [External deployment](https://docs.aws.amazon.com/AmazonECS/latest/userguide/scheduling_tasks.html)

Quest'opzione permette l'utilizzo di un deployment controller di terze parti in modo da avere il pieno controlo sul processo di rilascio. Ciò è possibile tramite l'interazione con le API di gestione del service o del task set.

Ci sono però però degli svantaggi, tra cui l'impossbilità di utilizzare il service autoscaling.

### Load Balancing

//source: [Service load balancing](https://docs.aws.amazon.com/AmazonECS/latest/userguide/scheduling_tasks.html)

Utilizzando un service in accoppiata con un Elastic Load Balancer, le richieste ricevute dal load balancer verranno distribuite sui task presenti nel servizio. I service che utilizzano Fargate supportano sia Application Load Balancer che Network Load Balancer.
Gli Application Load Balancer in particolare offrono alcune caratteristiche interesanti, tra cui la possibilità per ogni service di servire traffico da più load balancer.

## Quotas

//source: [Amazon ECS service quotas](https://docs.aws.amazon.com/AmazonECS/latest/userguide/service-quotas.html)

|                               | Quota   | Soft-Limit | Commenti   |
| ----------------------------- | ------- | ---------- | ---------- |
| Cluster                       | 10000   | Sì         | per Region |
| Service per cluster           | 5000    | Sì         |            |
| Task per service              | 5000    | Sì         |            |
| Revisioni per task definition | 1000000 | No         |            |
| Dimensione task definition    | 64KiB   | No         |            |
| Container per task definition | 10      | No         |            |
| Target groups per service     | 10      | No         |            |
