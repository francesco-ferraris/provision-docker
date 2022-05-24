[![CI](https://app.travis-ci.com/francesco-ferraris/provision-docker.svg?branch=travis-ci-dev)](https://app.travis-ci.com/francesco-ferraris/provision-docker)
# provision-docker

## Obiettivo

Questo playbook implementa l'esecuzione delle seguenti attività su Google Cloud Platform:
1. Provisioning di due VM CentOS. Le VM possono essere locali oppure su un
Cloud provider a scelta.
2. Configurare le VM:
   1. Assicurarsi che la partizione utilizzata da Docker abbia almeno 40GB di
   spazio disponibile, effettuando un opportuno resize.
3. Setup di Docker sulle VM
4. Configurare Docker:
   1. Esporre le API REST del Docker Daemon in modo sicuro
   2. Assicurarsi che il Docker Daemon sia configurato come un servizio che
   parta automaticamente all'avvio del sistema
5. Configurare un Docker Swarm sulle VM, che sia accessibile in modo sicuro.
Assicurarsi di riuscire ad interagire e deployare servizi sullo Swarm dalla
macchina locale.
6. Testare un task a scelta dei precedenti utilizzando Molecule
   1. Il task scelto è il numero 3.

Per la parte di Continuous Integration sono stati realizzate le seguenti attività con Travis CI:
1. Configurare una pipeline di Continuous Integration su un tool a scelta
2. La pipeline deve: 
   1. Eseguire il linting del codice e fallire in caso di errori, che vanno
   opportunamente corretti
3. Eseguire il test realizzato al punto 6 in automatico ad ogni push di codice
sul repository

## Struttura

Ogni attività è stata realizzata definendo uno specifico ruolo che implementa la feature richiesta.
All'interno della cartella _roles_ sono disponibili i ruoli che implementano le rispettive attività:
1. create_gcp_centos_instances
2. configure_instances
3. install_docker
4. configure_docker
5. configure_docker_swarm

Il punto 6 è implementato con Molecule all'interno del ruolo _install_docker_.

### create_gcp_centos_instances

Il ruolo utilizza i moduli ansible specifici per la piattaforma GCP per creare le macchine
virtuali su network di default con l'opzione per utilizzare un IP pubblico. Le macchine vengono successivamente aggiunte
ad un inventario runtime.
Le macchine virtuali sono configurate con due dischi e l'autenticazione alla piattaforma avviene con
in service account.
Variabili configurabili:

      instance_name: test # Nome dell'istanza
      image_version: centos-7-v20220406 # Versione immagine
      machine_type: n1-standard-1 # Taglio dell'istanza
      boot_disk_size_gb: 50 # Dimensione boot disk
      data_disk_size_gb: 100 # Dimensione data disk
      disk_type: pd-standard # Tipologia di disco
      inventory_group: centos_instances # Gruppo dell'inventario dinamico
      public_ip: false # Creazione istanza con IP pubblico
      region: europe-west4 # Regione di deployment
      zone: europe-west4-a # Zona di deployment
      project: project-id
      auth_kind: serviceaccount
      service_account_file: ~/service-account-key.json
      ssh_key_filename: devops-key # Nome chiave usata da Ansible per operare sull'istanza
      ssh_user: devops # Utenza usata da Ansible per operare sull'istanza

### configure_instances
Il ruolo configura le istanze create allocando una partizione LVM da 40 GB a docker.
Variabili configurabili:

      device: /dev/sdb # Device utilizzato da Docker
      docker_mountpath: /var/lib/docker # Mountpoint utilizzabile da Docker

### install_docker
Il ruolo Esegue l'installazione di docker CE sulle istanze facendo uso del ruolo _geerlingguy.docker_
disponibile su ansible-galaxy.
Non sono state apportate personalizzazione particolari al ruolo dal momento che effettua un'installazione
corretta del software e l'attività non prevede requisiti particolari.
Variabili configurabili:

      edition: ce # Edizione di Docker
      version: 20.10.* # Versione di Docker
      users: # Utenze abilitata a usare Docker
        - devops

### configure_docker
Il ruolo effettua la configurazione di docker sulle istanze riutilizzando e adattando il ruolo
_alexinthesky.secure-docker-daemon_. Le personalizzazioni effettuate si assicurano che la libreria 
_openssl_ sia installata sulle istanze (necessaria per la generazione dei certificati), configurano
il service docker per l'avvio automatico ad ogni startup dell'istanza e configurano il docker affinché
sia esposto in maniera sicura con i certificati generati.
Variabili configurabili:

      docker_swarm_addr: "{{ hostvars[inventory_hostname]['ansible_' + docker_swarm_interface]['ipv4']['address'] }}" # Endpoint
      docker_swarm_interface: eth0 # Interfaccia di rete livello 2
      docker_swarm_managers_ansible_group: docker-swarm-managers # Nome del gruppo contenente i manager del cluster
      docker_swarm_port: "2377" # Porta utilizzata da Docker Swarm per l'interazione
      docker_swarm_primary_manager: "{{ groups[docker_swarm_managers_ansible_group][0] }}" # Manager principale del cluster
      docker_swarm_workers_ansible_group: docker-swarm-workers # Nome del gruppo contenente i worker del cluster

### configure_docker_swarm
Il ruolo crea uno Swarm sulle due istanze: un manger e un worker.
Il task è stato implementato riadattando il ruolo _mrlesmithjr.ansible-docker-swarm_
inserendo le personalizzazioni per poter interagire con il docker esposto in modo sicuro sulle istanze
Il docker swarm viene configurato partendo da due gruppi all'interno dell'inventario runtime: un gruppo per
i manager e uno per i worker e i relativi task vengono eseguiti differenziando tra queste due
tipologie di istanze.
Variabili configurabili:

      docker_swarm_addr: "{{ hostvars[inventory_hostname]['ansible_' + docker_swarm_interface]['ipv4']['address'] }}" # Endpoint
      docker_swarm_interface: eth0 # Interfaccia di rete livello 2
      docker_swarm_managers_ansible_group: docker-swarm-managers # Nome del gruppo contenente i manager del cluster
      docker_swarm_port: "2377" # Porta utilizzata da Docker Swarm per l'interazione
      docker_swarm_primary_manager: "{{ groups[docker_swarm_managers_ansible_group][0] }}" # Manager principale del cluster
      docker_swarm_workers_ansible_group: docker-swarm-workers # Nome del gruppo contenente i worker del cluster

## Testing
E' stato realizzato un test con Molecule per il ruolo _install_docker_ che verifica la versione installata
di Docker. Lo scenario realizzato fa uso della libreria python _molecule_gce_ che permette 
la creazione di un ambiente su GCP su cui è eseguire i test.

## Continuous Integration
E' stata realizzata una pipeline con Travis CI che esegue il linting dei file yml con _ansible-lint_.
Ad ogni push la pipeline viene eseguita e fallisce in caso di errori su almeno un file.
Il linting è effettuato sia sui playbook che sui ruoli.
L'unico errore che viene ignorato è quello relativo ai FQCN.

Il test indicato al punto 6 viene eseguito ad ogni push del codice dalla pipeline e utilizza
un service account cifrato, che viene decriptato in fase di esecuzione della pipeline, per
connettersi con l'ambiente di test.

## Esecuzione
Realizzato con Ansible core 2.12.6

Senza un service account valido con le permission necessarie non è possibile eseguire il playbook (per semplicità ne ho usato uno con permessi di Editor).

Prima di eseguire il playbook è necessario configurare un ambiente virtuale
python e installare le dipendenze con questi comandi:

    ansible-galaxy install -r requirements.yml

Installare le dipendenze python:

    pip install -r requirements.txt

Eseguire il playbook:

    ansible-playbook playbook.yml -K --private-key <PERCORSO_PRIVATE_KEY>

Il percorso da inserire è quello dove verrà creata la chiave privata per connettersi alle istanze
create dal primo task (di default ~/.ssh/devops-key).
