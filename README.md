# [Kirahire](#kirahire)

Kirahire consente di effettuare il deploy di un cluster Docker Swam secondo le indicazioni fornite da [Kiratech](https://www.kiratech.it/).


## CI/CD pipeline
![CI/CD pipeline](https://github.com/pandvan/kirahire/actions/workflows/molecule.yml/badge.svg)


# Utilizzo

## Prerequisiti

I playbook Ansible per il deploy di Kirehire prevedono l'utilizzo di AWS.
Necessario:
1. creare almeno un utente IAM con accesso programmatico (access key/secret key).
1. creare una coppia di chiavi SSH


## Installazione requisiti
Si consiglia di installare le componenti su un virtual environment Python.

```bash
python3 -m venv <path_to_venv>
source <path_to_venv>/bin/activate
```

Aggiornare il gestore di pacchetti Python `pip`

```bash
pip install --upgrade pip
```

Quindi installare i requisiti, tra cui Ansible e i suoi ruoli e collezioni.

```bash
pip install --requirement requirements.txt
ansible-galaxy install --role-file requirements.yml
```


## Preparazione ambiente
Importare le variabili d'ambiente per consentire l'accesso al proprio account AWS

```bash
export AWS_ACCESS_KEY_ID=xyz
export AWS_SECRET_ACCESS_KEY=xyz
export AWS_REGION=xyz
```

In alternativa utilizzare il file `set_env.sh.example` come base por poi caricare le variabili nella shell corrente

```bash
cp set_env.sh.example set_env.sh
< modificare set_env.sh>
source set_env.sh
```


## Configurazione inventario
Il file `inventory/inventory.yml` contiene i riferimenti delle macchine che verranno create e il loro ruolo.
Fare riferimento al file fornito nel repository per adattare secondo proprie esigenze.

Le variabili che veicolano il comportamento dei playbook sono per lo più definite nel file `inventory/group_vars/docker_swarm.yml`.

Verificare di configurare come desiderato almeno le seguenti variabili

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


## Accesso al cluster Docker Swarm
Al termine del deploy vengono create i cerificati necessari a collegarsi alle API di Docker via TLS all'interno della cartella `/data/volumes/certs/client/*.pem` del nodo master.

Copiare tutti cerificati (`ca.pem`, `cert.pem`, `key.pem`) all'interno della cartella `.tls`

```bash
scp -i .ssh/<key.pem> 'ec2-user@e<EC2_public_DNS_name>:/data/docker-tls/volumes/certs/client/*.pem' .tls/
```

Verificare la connettività con le API del demone Docker

```bash
docker \
    --tlscacert=.tls/ca.pem \
    --tlscert=.tls/cert.pem \
    --tlskey=.tls/key.pem \
    -H=kirahire.example.io:2376 info
```

A titolo di exempio è presente un file Docker Compose in `example/docker-compose.yml` per testare il deploy di un servizio del cluster.

```bash
docker \
    --tlscacert=.tls/ca.pem \
    --tlscert=.tls/cert.pem \
    --tlskey=.tls/key.pem \
    -H=kirahire.example.io:2376 stack deploy -c example/docker-compose.yml test
```
