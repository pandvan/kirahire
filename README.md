# [Kirahire](#kirahire)

Kirahire consente di effettuare il deploy di un cluster Docker Swarm secondo le indicazioni fornite da [Kiratech](https://www.kiratech.it/) in forma di challenge nel processo di selezione.

L'automazione è definita mediante [Ansible](https://www.ansible.com/) con l'impiego di collezioni e ruoli di terze parti resi disponibili dalla comunità.


## CI/CD pipeline
Esito dell'ultima esecuzione della pipeline

![CI/CD pipeline](https://github.com/pandvan/kirahire/actions/workflows/molecule.yml/badge.svg)



# Utilizzo

## Prerequisiti

I playbook Ansible per il deploy di Kirahire assumo l'utilizzo del fornitore cloud AWS.
Prima di eseguirli, si rende necessario:
1. Identificare l'account AWS che si vuole utilizzare.
1. Creare almeno un utente IAM con accesso programmatico (access key + secret key).
1. Creare una coppia di chiavi SSH e avere a disposizione la relativa chiave privata.


## Installazione dipendenze
Si consiglia di operare e quindi installare le componenti su un virtual environment Python dedicato.

```bash
python3 -m venv <path_to_venv>
source <path_to_venv>/bin/activate
```

Una volta caricato il virtual environment, procedere con l'aggiornamento del gestore di pacchetti Python `pip`

```bash
pip install --upgrade pip
```

Quindi installare i moduli Python necessari, tra cui `ansible-core`. A seguito installare tutti i ruoli e collezioni necessari al funzionamento dei playbook mediante `ansible-galaxy`.

```bash
pip install --requirement requirements.txt
ansible-galaxy install --role-file requirements.yml
```


## Preparazione ambiente
Impostare le variabili d'ambiente per consentire l'accesso al proprio account AWS

```bash
export AWS_ACCESS_KEY_ID=xyz
export AWS_SECRET_ACCESS_KEY=xyz
export AWS_REGION=xyz
```

In alternativa è possibil utilizzare il file `set_env.sh.example` come base por poi caricare le variabili nella shell corrente

```bash
cp set_env.sh.example set_env.sh
# -- edit set_env.sh ---
source set_env.sh
```


## Configurazione inventario
Il file `inventory/inventory.yml` contiene i riferimenti delle macchine che verranno create e il loro ruolo (manager o worker del cluster Docker Swarm).
Fare riferimento al file fornito nel repository e adattarlo secondo proprie esigenze.

Le variabili che veicolano il comportamento dei playbook sono per lo più definite nel file `inventory/group_vars/docker_swarm.yml`.

Accertarsi di configurare in modo opportuno almeno le seguenti variabili

```
# ID del VPC
ec2_vm_vpc_id: vpc-xyz

# Network della subnet
ec2_vm_subnet_cidr: 172.31.240.0/28

# Nome della chiave SSH creata in precedenza
ec2_vm_ssh_key_name: kirahire

# Tipo di istanza EC2
ec2_vm_instance_type: t3a.micro

# ID dell'immagine per l'istanza EC2
ec2_vm_image_id: ami-0c1dd6e732dee7221
```


## Deploy
Copiare la chiave privata della coppia chiavi SSH creata in precedenza all'interno della cartella `.ssh`.

```bash
cp <path_to_priv_key> .ssh/kirahire.pem
```

Eseguire i playbook Ansible

```bash
ansible-playbook -i inventory/ 01_infra.yml
ansible-playbook -i inventory/ 02_setup.yml
```

I playbook contengono alcuni tag utili nel caso si voglia eseguire solo una parte dei task definiti

```bash
ansible-playbook -i inventory 02_setup.yml --list-tasks
ansible-playbook -i inventory 02_setup.yml --list-tags
ansible-playbook -i inventory 02_setup.yml --tags role::docker-swarm,role::docker_tls
```


## Accesso al cluster Docker Swarm
Al termine del deploy vengono creati i cerificati necessari a collegarsi alle API di Docker via TLS all'interno della cartella `/data/volumes/certs/client/*.pem` del nodo master.

Copiare tutti cerificati presenti (`ca.pem`, `cert.pem`, `key.pem`) all'interno della cartella `.tls`

```bash
mkdir .tls
scp -i .ssh/kirahire.pem 'ec2-user@e<EC2_public_DNS_name>:/data/docker-tls/volumes/certs/client/*.pem' .tls/
```

Dal momento che i playbook non gestiscono la registrazione di un nome DNS pubblico per il nodo master, per i test iniziali è necessario modificare manualmente il profilo file `/etc/hots/` aggiungendo la riga

```
[...]
# IP pubblico nodo master / nome scelto per la creazione del certificato
150.100.101.102 kirahire.example.io
```

Verificare la connettività con le API del demone Docker

```bash
docker \
    --tlscacert=.tls/ca.pem \
    --tlscert=.tls/cert.pem \
    --tlskey=.tls/key.pem \
    -H=kirahire.example.io:2376 info
```

A titolo di exempio è presente un file Docker Compose in `example/docker-compose.yml` per testare il deploy di un servizio nel cluster.

```bash
docker \
    --tlscacert=.tls/ca.pem \
    --tlscert=.tls/cert.pem \
    --tlskey=.tls/key.pem \
    -H=kirahire.example.io:2376 stack deploy -c example/docker-compose.yml test
```
