## Portainer auf Linux
*von Silas Forrer*

# Was ist Portainer?
    Portainer ist eine schlanke Verwaltungsoberfläche, mit der Sie Ihre verschiedenen Docker-Umgebungen einfach verwalten können.

    Portainer bietet eine leichte und einfache Lösung für die Verwaltung von Docker-Containern und Swarm-Diensten über eine Weboberfläche. \
    Portainer unterstützt eine breite Palette von Funktionen für die Verwaltung von Docker-Containern, wie z. B. die Verwaltung oder der Erstellung \
    und Löschung von Swarm-Diensten, Benutzerauthentifizierung, Berechtigungen, Verbindungsaufbau, Ausführung von Befehlen in der Konsole von \
    laufenden Containern und die Anzeige von Container-Logs.


# Inhaltsverzeichnis

- **[Darstellung](#Darstellung)**
- **[Umgebung](#Umgebung)**
  - [Verwendete Software](#verwendete-software)
  - [VM-Spezifikationen](#vm-spezifikationen)
- **[Sourcecode](#Sourcecode)**
  - [vagrantfile](#vagrantfile)
  - [Basis Konfiguration](#basis-konfiguration)
  - [Konfigurationen durch die Shell](#konfigurationen-durch-die-shell)
    - [Initial Tasks](#initial-tasks)
    - [Firewall](#firewall)
    - [Portainer Docker-Container](#portainer-docker-container)
    - [Abschluss](#abschluss)
- **[Testing](#Testing)**
- **[Fazit](#Fazit)**
- **[Quellen](#Quellen)**

# Darstellung

![Struktur](image/umgebung.png)

# Umgebung
## Verwendete Software:

| **Produkt** | **Version** |
----------|----------
| Vagrant | 2.2.19 |
| VirtualBox | 6.1 |
| Portainer | 2.9.3 |
| Ubuntu | 20.04 |
  
## VM-Spezifikationen:

| **Attribut** | **Wert** |
-------|---------------
| IPv4 | 192.169.4.20 |
| CPU-Cores | 5 |
| Memory | 5GB |
| USB | off |
| USBehci | off |

# Sourcecode

## Vagrantfile

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"
config.vm.hostname = "apache"
config.vm.network "private_network", ip: "192.169.4.20"
config.vm.provider "virtualbox" do |vb|
  vb.name = "Apache in Docker"
  vb.customize ["modifyvm", :id, "--cpus", "5"]
  vb.customize ["modifyvm", :id, "--memory", "5000"]
  vb.customize ["modifyvm", :id, "--usb", "off"]
  vb.customize ["modifyvm", :id, "--usbehci", "off"]
end
config.vm.provision "shell", inline: <<-SHELL
  sudo su - &&
  apt-get update -y &&
  apt-get upgrade -y && 
  apt-get autoremove -y &&
  apt-get install docker docker-compose -y &&

  systemctl enable docker &&
  systemctl start docker &&
  
  apt-get install ufw -y &&
  apt-get install openssh-server -y &&
  ufw enable &&
  ufw allow 9009 &&
  ufw allow 22 &&
  
  docker volume create portainer_volume &&
  docker run -d -p 8008:8000 -p 9009:9443 --name portainer_container \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_volume:/data \
  portainer/portainer-ce:2.9.3 &&

  apt-get autoremove -y &&
  apt-get autoclean -y
SHELL
end
```

## Basis Konfiguration

Zu Begin wird im Vagrantfile die grundsätzlichen Informationen, \
wie zum Beispiel Netzwerkkonfigurationen (IP), Hardware (Memory, CPU etc.), das Gastbetriebssystem etc.

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"
  config.vm.hostname = "apache"
  config.vm.network "private_network", ip: "192.169.4.20"
  config.vm.provider "virtualbox" do |vb|
    vb.name = "Apache in Docker"
    vb.customize ["modifyvm", :id, "--cpus", "5"]
    vb.customize ["modifyvm", :id, "--memory", "5000"]
    vb.customize ["modifyvm", :id, "--usb", "off"]
    vb.customize ["modifyvm", :id, "--usbehci", "off"]
  end
```

## Konfigurationen durch die Shell
> Um Shell/Bash-Commands zu nutzen muss zuerst ein Bereich folgendermassen definiert werden:
> ```ruby
> config.vm.provision "shell", inline: <<-SHELL
> ```
> \
> **(?)** Die Bezeichnung `&&` am Ende jeder Zeile macht, 
> dass die nächste zeile nur dann ausgeführt wird wenn der command auf der aktuellen Zeile ohne Error beendet wurde.

### Initial Tasks
Als erstes wechsle ich den Nutzer zu "root", damit ich keine Probleme mit Berechtigungen bekomme 
und nicht immer "sudo" schreiben muss. \
Als zweites aktualisiere und bereinige ich das System, jeweils mit einem "-y" um Benutzerinteraktionen zu umgehen. \
Sobald alles aktuell ist wird [docker](https://www.docker.com/) und [docker-compose](https://docs.docker.com/compose/) \ installiert.

```bash
sudo su - &&
apt-get update -y &&
apt-get upgrade -y && 
apt-get autoremove -y &&
apt-get install docker docker-compose -y &&
```

Sobald die Dockerprodukte installiert sind wird eingestellt, \
dass das diese bei jedem Systemstart automatisch auch starten.

```bash
systemctl enable docker &&
systemctl start docker &&
```

### Firewall
Als Nächstes wird die weitverbreitete Firewall [ufw](https://wiki.ubuntuusers.de/ufw/) installiert. \
Sobald diese problemlos installiert wurde wird der Port 9009, welcher für das Portainer-Webgui genutzt wird, Zugelassen. \
Für den SSH zugriff muss noch die Software [Openssh-server](https://ubuntu.com/server/docs/service-openssh) installiert und der Port 22 Zugelassen werden.

```bash
apt-get install ufw -y &&
apt-get install openssh-server -y &&
ufw enable &&
ufw allow 9009 &&
ufw allow 22 &&
```

### Portainer Docker-Container
Vor dem Starten des Cotainers, wird ein [Dockervolume](https://docs.docker.com/storage/volumes/#create-and-manage-volumes) erstellt, \
wo später die Dateien des containers abgelegt werden, sodass diese beim stoppen des Containers nicht verloren gehen. \
Nun wird das [Portainer](https://docs.portainer.io/v/ce-2.9/start/install/server/docker/linux#deployment) Dockercontainerimage gestartet. \
Folgende [Docker run Container-Parameter](https://docs.docker.com/engine/reference/commandline/run/#options) wurden verwendet: \
| Parameter | Beschreibung |
------------|---------------
| `-d` | **_"detached"_** macht, dass der Container im Hintergrund läuft und nicht z.B. Ausgaben ins Linux-terminal schreibt |
| `-p` | **_"port"_** mit diesem Parameter können Container Ports an Host-Ports gebunden werden |
| `--name` | der name des Containers |
| `-v` | **_"volume"_** hier wird angegeben welcher Container-Pfad an welchen Host-Pfad gebunden wird |

```bash
docker volume create portainer_volume &&
docker run -d -p 8008:8000 -p 9009:9443 --name portainer_container \
--restart=always \
-v /var/run/docker.sock:/var/run/docker.sock \
-v portainer_volume:/data \
portainer/portainer-ce:2.9.3 &&
```

### Abschluss
> der "SHELL" Teil im Vagrantfile muss, wie im folgenden Code-Teil, abgeschlossen werden:
> ```ruby
>   SHELL
> end
> ```

Nun werden noch alle unnötigen Pakete vom System entfernt.

```bash
apt-get autoremove -y &&
apt-get autoclean -y
```


# Testing

Sobald das Vagrantfile, mit `vagrant up` gestartet wird, kann auf das [Portainer WebGUI](https://192.169.4.20:9009) zugegriffen werden.

Wenn die Installation erfolgreich war sollte nun eine Site zusehen sein (wie im unteren Bild), \
wo beim ersten Login die Admin-Benutzer informationen gesetzt werden müssen, das heisst Passwort und Nutzername.

![WebGUI](image/Loginscreen.png)


Als erstes sollte ein simpler Verbindungstest mit `ping` durchgeführt werden.

![Ping](image/Ping.png) 

Wie im Bild ersichtlich, war dieser Test erfolgreich.

Oder man könnte direkt das [Portainer WebGUI](https://192.169.4.20:9009) aufrufen:
![Portainer](image/Webgui.png)

Auch der Zugriff aufs WebGUI funktioniert. Somit können wir abschliessend sagen, dass unser System funktioniert.


# Fazit


# Quellen
[Markdown Cheatsheet](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet)

[Vagrant Website](https://www.vagrantup.com/)

[Vagrant Tutorial by Hashicorp](https://learn.hashicorp.com/tutorials/vagrant/getting-started-provisioning)

