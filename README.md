# Dokumentation: Einrichtung der VM und Ausführung des Ansible Playbooks

----
## Ordnerstruktur
```css
odoo-ansible-demo/
├── ansible/
│   ├── group_vars/
│   │   └── all.yml
│   ├── roles/
│   │   ├── helm-deploy/
│   │   │   ├── tasks/
│   │   │   │   ├── main.yml
│   │   │   ├── templates/
│   │   │   │   └── values.j2
│   │   │   ├── defaults/
│   │   │   │   └── main.yml
│   ├── playbooks/
│   │   └── deploy-odoo.yml
│   ├── inventory/
│   │   └── hosts.yml
│   └── ansible.cfg
├── README.md
```

## Voraussetzungen
1. Eine Ubuntu VM auf UTM ist bereits installiert.
2. MicroK8s ist installiert, aber noch nicht konfiguriert.
3. Helm 3 ist installiert.
4. Ansible ist installiert und eingerichtet auf deinem Host-MacBook.

---

## Schritt 1: Vorbereitung der Ubuntu-VM

### 1.1 MicroK8s Installation und Einrichtung
Führe die folgenden Befehle auf der VM aus:
```bash
# MicroK8s installieren
sudo snap install microk8s --classic

# Benutzer zur MicroK8s-Gruppe hinzufügen
sudo usermod -aG microk8s $USER

# Verzeichnisseinstellungen korrigieren
sudo chown -R $USER ~/.kube

# Sitzung neu laden, um Gruppenzugehörigkeit zu aktualisieren
newgrp microk8s

# MicroK8s-Add-ons aktivieren
microk8s enable dns storage helm3

# Status prüfen
microk8s status --wait-ready
```

### 1.2 Helm Repository hinzufügen
```bash
# Bitnami Helm Repository hinzufügen
microk8s helm3 repo add bitnami https://charts.bitnami.com/bitnami

# Helm Repository aktualisieren
microk8s helm3 repo update
```

### 1.3 Python-Umgebung einrichten
Das Ansible-Community-Modul für Kubernetes benötigt Python-Bibliotheken. Installiere diese auf der VM:
```bash
# Pip installieren (falls nicht vorhanden)
sudo apt update
sudo apt install -y python3-pip python3-pyyaml python3-requests

# Kubernetes-Modul installieren
pip3 install kubernetes --break-system-packages

# Installation überprüfen
python3 -c "import kubernetes; print(kubernetes.__version__)"
```

### 1.4 SSH-Zugriff einrichten
Damit Ansible auf die VM zugreifen kann:
```bash
# SSH-Schlüssel generieren (falls noch nicht vorhanden)
ssh-keygen -t rsa -b 2048 -f ~/.ssh/id_rsa -N ""

# Zeile Anpassen auf VM /etc/ssh/sshd_config
PubkeyAuthentication yes

# Öffentlichen Schlüssel zur VM hinzufügen
scp /path/to/id_rsa.pub user@<vm-ip>:~/.ssh

# Schlüssel auf VM zu authoized keys hinzufügen
  cat ~/.ssh/id_rsa.pub >> authorized_keys

# Verbindung testen
ssh -i /path/to/id_rsa user@<vm-ip>
```

---

## Schritt 2: Ansible Playbook ausführen

### 2.1 Inventory-Datei anpassen
Bearbeite die Datei `ansible/inventory/hosts.yml` und füge die IP-Adresse oder den Hostnamen der VM hinzu:
```yaml
all:
  hosts:
    ubuntu-vm:
      ansible_host: <vm-ip>
      ansible_user: <vm-user>
```

### 2.2 Playbook vorbereiten
Stelle sicher, dass das Playbook und die benötigten Dateien vorhanden sind. Navigiere in das Verzeichnis deines Projekts:
```bash
cd /path/to/microdogs
```

### 2.3 Playbook ausführen
Führe den folgenden Befehl aus, um das Playbook auszuführen:
```bash
ansible-playbook playbooks/deploy-odoo.yml
```

Während der Ausführung des Playbooks wird Ansible die notwendigen Schritte automatisieren, um das Odoo-Deployment auf MicroK8s bereitzustellen.

### 2.4 Ergebnisse prüfen
Prüfe nach der erfolgreichen Ausführung die laufenden Pods auf der VM:
```bash
microk8s kubectl get pods -n odoo
```

Um die Odoo-Anwendung zu erreichen, prüfe die Services:
```bash
microk8s kubectl get svc -n odoo
```
Standardmäßig wird ein NodePort-Service erstellt. Greife mit der IP der VM und dem angegebenen Port auf die Anwendung zu:
```
http://<vm-ip>:<nodeport>
```

---

## Schritt 3: Skalierung und Erweiterung

### 3.1 Weitere Deployments hinzufügen
- Bearbeite die Datei `inventory/hosts.yml`, um neue Deployments zu definieren in dem du die liste `odoo_deployments` ergänzts.
- Passe die Helm-Werte in der Datei `templates/values.j2` an, falls notwendig.

### 3.2 Deployment skalieren
Führe den folgenden Befehl aus, um die Anzahl der Pods zu erhöhen:
```bash
microk8s kubectl scale deployment odoo --replicas=3 -n odoo
```

## Allgemeines Kubernetes Handling

### Alias für microk8s Kommando
Da es nervig ist jedes mal `sudo microk8s` vor jedem kubectl Befehl einzugeben kannst du folgenden alias erstellen:
```bash
# Füge einen Alias hinzu, damit microk8s kubectl als kubectl ausgeführt wird.
echo 'alias kubectl="microk8s kubectl"' >> ~/.bashrc
# Lade die Konfiguration neu:
source ~/.bashrc
```

### Ressourcen löschen
Wenn du Deployments, StatefulSets oder andere Ressourcen entfernen möchtest, kannst du die folgenden Befehle verwenden:

#### a) Einzelne Deployments löschen
```bash
kubectl delete deployment <deployment-name> -n <namespace>
```
**Beispiel:**
```bash
kubectl delete deployment odoo1 -n default
```

#### b) StatefulSets löschen
```bash
kubectl delete statefulset <statefulset-name> -n <namespace>
```
**Beispiel:**
```bash
kubectl delete statefulset odoo-postgresql -n odoo2
```

#### c) Services löschen
```bash
kubectl delete service <service-name> -n <namespace>
```
**Beispiel:**
```bash
kubectl delete service odoo-postgresql-hl -n default
```

#### d) Namespace löschen
Falls ein Namespace nicht mehr benötigt wird, kannst du ihn komplett löschen. Dies entfernt alle Ressourcen darin:
```bash
kubectl delete namespace <namespace>
```
**Beispiel:**
```bash
kubectl delete namespace odoo2
```

### Überprüfung von Ressourcen
- Liste alle Ressourcen in einem Namespace auf:
  ```bash
  kubectl get all -n <namespace>
  ```

- Prüfe alle Ressourcen über alle Namespaces hinweg:
  ```bash
  kubectl get all -A
  ```

- Suche nach spezifischen Ressourcen (z. B. "odoo"):
  ```bash
  kubectl get all -A | grep odoo
  ```

