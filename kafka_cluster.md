# **1️⃣ Préparer la VM de base (Base VM)**

## **1. Installer Ubuntu + dépendances**

```bash
sudo apt update
sudo apt install openjdk-8-jdk openssh-server -y
java -version
```

### Ajouter JAVA_HOME

```bash
sudo nano /etc/environment
```

Ajouter :

```
JAVA_HOME="/usr/lib/jvm/java-8-openjdk-amd64"
PATH="$PATH:$JAVA_HOME/bin"
```

```bash
source /etc/environment
```

---

## **2. Télécharger et installer Kafka**

```bash
wget https://downloads.apache.org/kafka/3.7.1/kafka_2.12-3.7.1.tgz
sudo tar -xvzf kafka_2.12-3.7.1.tgz -C /opt/
sudo mv /opt/kafka_2.12-3.7.1 /opt/kafka
```

Créer les dossiers requis :

```bash
sudo mkdir -p /var/lib/kafka/logs
sudo mkdir -p /var/lib/zookeeper
sudo mkdir -p /opt/kafka/logs
sudo chown -R $USER:$USER /var/lib/kafka /var/lib/zookeeper /opt/kafka/logs
```

---

# **2️⃣ Configurer le réseau (VM de base)**

## **Activer Adapter 1 = NAT**

## **Activer Adapter 2 = Host-Only (ex: 192.168.56.x)**

### Configurer l’IP statique

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Exemple :

```yaml
network:
  version: 2
  ethernets:
    enp0s8:
      addresses:
        - 192.168.56.101/24
      gateway4: 192.168.56.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

Appliquer :

```bash
sudo netplan apply
ip addr
```

---

# **3️⃣ Nettoyer la VM avant clonage**

```bash
sudo rm -rf /tmp/kafka-logs
sudo rm -f /var/lib/kafka/logs/meta.properties
```

### **Éteindre la VM ⇒ Cloner en 3 machines**

* kafka-node-1
* kafka-node-2
* kafka-node-3

**FULL clone + new MAC addresses**

---

# **4️⃣ Configurer chaque nœud après clonage**

## 4.1 Assigner une IP statique différente dans chaque VM

Exemples :

* Node1 → 192.168.56.101
* Node2 → 192.168.56.102
* Node3 → 192.168.56.103

Mettre à jour `/etc/hostname`, `/etc/hosts`

---

# **5️⃣ Configurer Zookeeper sur chaque VM**

### Modifier `/opt/kafka/config/zookeeper.properties` :

```
tickTime=2000
initLimit=10
syncLimit=5

dataDir=/var/lib/zookeeper
dataLogDir=/opt/kafka/logs

clientPort=2181

server.1=192.168.1.189:2888:3888
server.2=192.168.1.120:2888:3888
server.3=192.168.1.108:2888:3888

```

### Définir le myid

Node1 :

```bash
echo 1 > /var/lib/zookeeper/myid
```

Node2 :

```bash
echo 2 > /var/lib/zookeeper/myid
```

Node3 :

```bash
echo 3 > /var/lib/zookeeper/myid
```

---

# **6️⃣ Configurer Kafka sur chaque nœud**

Modifier `/opt/kafka/config/server.properties` :

### Node 1 :

```
broker.id=1
listeners=PLAINTEXT://192.168.56.101:9092
advertised.listeners=PLAINTEXT://192.168.56.101:9092
log.dirs=/var/lib/kafka/logs
zookeeper.connect=192.168.56.101:2181,192.168.56.102:2181,192.168.56.103:2181
```

### Node 2 :

(modifier l’IP et broker.id = 2)

### Node 3 :

(modifier l’IP et broker.id = 3)

---

# **7️⃣ Démarrer le cluster**

## Démarrer Zookeeper sur chaque VM :

```bash
/opt/kafka/bin/zookeeper-server-start.sh /opt/kafka/config/zookeeper.properties
```

## Démarrer Kafka :

```bash
/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
```

---

# **7️⃣ bis — (Optionnel mais Recommandé) : Créer des services systemd**

Cette section permet d'avoir un cluster **professionnel**, avec :

* redémarrage automatique
* démarrage au boot
* gestion via `systemctl`

---

## **Créer le service Zookeeper**

```bash
sudo nano /etc/systemd/system/zookeeper.service
```

Ajouter :

```
[Unit]
Description=Apache Zookeeper Server
Documentation=http://zookeeper.apache.org
After=network.target

[Service]
Type=simple
ExecStart=/opt/kafka/bin/zookeeper-server-start.sh /opt/kafka/config/zookeeper.properties
ExecStop=/opt/kafka/bin/zookeeper-server-stop.sh
Restart=on-abnormal

[Install]
WantedBy=multi-user.target
```

Activer le service :

```bash
sudo systemctl daemon-reload
sudo systemctl enable zookeeper
sudo systemctl start zookeeper
sudo systemctl status zookeeper
```

---

## **Créer le service Kafka**

```bash
sudo nano /etc/systemd/system/kafka.service
```

Ajouter :

```
[Unit]
Description=Apache Kafka Server
Documentation=http://kafka.apache.org/documentation.html
After=network.target zookeeper.service

[Service]
Type=simple
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh
Restart=on-abnormal

[Install]
WantedBy=multi-user.target
```

Activer :

```bash
sudo systemctl daemon-reload
sudo systemctl enable kafka
sudo systemctl start kafka
sudo systemctl status kafka
```

---

# **8️⃣ Vérifier le cluster**

## Vérifier le contrôleur :

```bash
/opt/kafka/bin/zookeeper-shell.sh 192.168.56.101:2181
get /controller
```

## Vérifier la connectivité :

```bash
nc -vz 192.168.56.101 9092
nc -vz 192.168.56.101 2181
```

---

# **9️⃣ Tester Kafka (topic, producer, consumer)**

## Créer un topic répliqué :

```bash
/opt/kafka/bin/kafka-topics.sh --create \
  --topic test-topic \
  --partitions 3 \
  --replication-factor 2 \
  --bootstrap-server 192.168.56.101:9092
```

## Vérifier :

```bash
/opt/kafka/bin/kafka-topics.sh --describe \
  --topic test-topic \
  --bootstrap-server 192.168.56.101:9092
```

## Producer :

```bash
/opt/kafka/bin/kafka-console-producer.sh \
  --topic test-topic \
  --bootstrap-server 192.168.56.101:9092
```

## Consumer :

```bash
/opt/kafka/bin/kafka-console-consumer.sh \
  --topic test-topic \
  --from-beginning \
  --bootstrap-server 192.168.56.101:9092
```

---

# **🔟 Tester la tolérance aux pannes**

### 1. Stopper un broker (ex : node 1)

```bash
sudo systemctl stop kafka
```

### 2. Vérifier le nouveau leader

```bash
/opt/kafka/bin/kafka-topics.sh --describe \
  --topic test-topic \
  --bootstrap-server 192.168.56.102:9092
```

### 3. Relancer le broker et vérifier ISR

---
