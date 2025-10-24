# Guía de Rol: PROXY + MONITOREO (Tu Rol)

Esta guía ha sido corregida y aclarada: se han uniformado los pasos, corregido ejemplos de configuración (Netplan, Nginx, Prometheus y MariaDB), adaptado el SQL a MariaDB y mejoradas las instrucciones de verificación y hardening. Reemplaza todos los valores entre `<>` por los valores reales de tu entorno (por ejemplo `<VLAN_ID>`, `<IP_PROXY>`, `<IP_APP1>`, `<IP_APP2>`, `<IP_DB>`, `<IP_GATEWAY>`).

## Información general

- IP de gestión/salto: `192.168.100.219`
- IP de servicio (VLAN): `<IP_PROXY>` (ej. `192.168.101.2`)
- Puertos relevantes:
  - Nginx (HTTP): `80`
  - Grafana: `3000`
  - Prometheus: `9090`
  - Node Exporter: `9100`
  - App Node: `3000`
  - MariaDB: `3306`

---

## Fase 0 — Acceso, reconocimiento e instalación

1. Accede a tu VM:
   - Conéctate al Jump Server: `ssh adminsrv@jhoelvillca`
   - Desde allí salta a tu VM: `ssh tu-usuario@192.168.100.219`

2. Reconocimiento básico:
   - `ip a`                 ← identifica la interfaz (probablemente `ens18`)
   - `sudo ufw status`      ← confirma estado del firewall
   - `ping -c 3 8.8.8.8`    ← confirma conectividad a Internet

3. Instalación de software (ejemplo Ubuntu/Debian):
```bash
sudo apt update
sudo apt install -y nginx prometheus prometheus-node-exporter grafana
sudo systemctl enable --now grafana-server prometheus-node-exporter
```
Nota: los nombres de paquete pueden variar según la distro. Ajusta si es necesario.

---

## Fase 1 — Configuración de red (Netplan)

1. Haz copia de seguridad y abre el archivo:
```bash
sudo cp /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.bak
sudo nano /etc/netplan/50-cloud-init.yaml
```

2. Ejemplo de configuración netplan con VLAN. Reemplaza los placeholders:
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens18:
      addresses:
        - "192.168.100.219/24"
      routes:
        - to: "default"
          via: "192.168.100.1"
      nameservers:
        addresses: [8.8.8.8]

  vlans:
    vlan<VLAN_ID>:            # e.g. vlan101
      id: <VLAN_ID>          # e.g. 101
      link: ens18
      addresses:
        - "<IP_PROXY>/29"    # e.g. "192.168.101.2/29"
      nameservers:
        addresses: [8.8.8.8]
      routes:
        - to: default
          via: "<IP_GATEWAY>" # e.g. "192.168.101.1"
```
- Asegúrate de mantener la indentación YAML correcta.
- Sustituye `<VLAN_ID>`, `<IP_PROXY>` y `<IP_GATEWAY>`.

3. Aplica y prueba:
```bash
sudo netplan apply
ping -c 3 <IP_GATEWAY>
ip a    # verifica que la interfaz vlan exista (p.ej. vlan101)
```

---

## Fase 2 — Configuración de servicios

### 1) Nginx (balanceador)

Edita el sitio por defecto:
```bash
sudo nano /etc/nginx/sites-available/default
```

Ejemplo de configuración (reemplaza IPs de apps):
```nginx
upstream loadbalancer {
    server <IP_APP1>:3000;
    server <IP_APP2>:3000;
}

server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://loadbalancer;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Prueba y recarga:
```bash
sudo nginx -t && sudo systemctl reload nginx
```

### 2) Prometheus (colector)

Edita `/etc/prometheus/prometheus.yml` y añade (o integra) un job para `node_exporter`:
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "node_exporter"
    static_configs:
      - targets:
        - "<IP_PROXY>:9100"
        - "<IP_APP1>:9100"
        - "<IP_APP2>:9100"
        - "<IP_DB>:9100"
```
- Reemplaza los placeholders por las IPs reales.
- Si Prometheus corre en otra máquina, ajusta la URL desde Grafana.

Reinicia Prometheus:
```bash
sudo systemctl restart prometheus.service
```

### 3) Grafana

- Grafana normalmente escucha en `:3000`.
- Accede en `http://<IP_PROXY>:3000` (o usando el nombre DNS que provea el lab).
- Usuario/Password por defecto: `admin`/`admin` (se te pedirá cambiar al primer login).
- Añade Data Source:
  - Tipo: Prometheus
  - URL: `http://<IP_PROXY>:9090`
- Importa el dashboard ID `10242` (o el que indique la práctica), y selecciona el Data Source.

---

## Fase 3 — Hardening (UFW)

Reglas recomendadas:
```bash
sudo ufw allow ssh
sudo ufw allow 80/tcp     # Nginx
sudo ufw allow 3000/tcp   # Grafana
sudo ufw allow 9090/tcp   # Prometheus (si debe ser accesible)
sudo ufw allow 9100/tcp   # Node Exporter (si debe ser accesible)
sudo ufw enable
```
Considera limitar acceso a `9090`/`3000`/`9100` solo desde subredes de gestión o desde IPs de alumnos/Proxy según la política del laboratorio.

---

## Fase 4 — Verificación

1. Verifica balanceo en Nginx accediendo a `http://<IP_PROXY>/` (puede mostrar 502 hasta que las apps estén levantadas).
2. Verifica Prometheus:
   - Targets: `http://<IP_PROXY>:9090/targets`
3. Verifica Grafana y que el dashboard muestre métricas.
4. Comprueba que `node_exporter` esté activo en cada host:
```bash
sudo systemctl status prometheus-node-exporter
curl http://localhost:9100/metrics | head
```

---

# Guía de Rol: APLICACIÓN 1 (Alumno 2)

- IP de gestión: `192.168.100.213`
- IP de servicio (VLAN): `<IP_APP1>` (ej. `192.168.101.3`)

Fases: siguen los mismos pasos que el Proxy en cuanto a Netplan y UFW, con particularidades:

Fase 0 — Instalar software:
```bash
sudo apt update
sudo apt install -y prometheus-node-exporter
# Node.js env
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
. "$HOME/.nvm/nvm.sh"
nvm install 22
npm install -g pm2
sudo systemctl enable --now prometheus-node-exporter
```

Fase 1 — Netplan: añade la sección vlan con `<IP_APP1>` (ver ejemplo anterior).

Fase 2 — Clonar y configurar la app:
```bash
git clone https://github.com/marceloquispeortega/api-restful-crud-movies app
cd app
npm install
cp .env.example .env
nano .env   # ajustar DB_HOST, etc.
```
En `.env`:
```
PORT=3000
DB_HOST='<IP_DB>'
```
Levanta la app con PM2:
```bash
pm2 start app.js --name app1
```

Fase 3 — UFW:
```bash
sudo ufw allow ssh
sudo ufw allow 3000/tcp
sudo ufw allow 9100/tcp
sudo ufw enable
```

Fase 4 — Avisar al Proxy que estás online (o registrar en Prometheus target).

---

# Guía de Rol: APLICACIÓN 2 (Alumno 3)

Sigue los pasos de Aplicación 1 reemplazando las IPs por `<IP_APP2>` y arrancando con:
```bash
pm2 start app.js --name app2
```

---

# Guía de Rol: BASE DE DATOS (Alumno 4)

- IP de gestión: `192.168.100.206`
- IP de servicio (VLAN): `<IP_DB>` (ej. `192.168.101.5`)

Fase 0 — Instalación:
```bash
sudo apt update
sudo apt install -y mariadb-server prometheus-node-exporter
sudo systemctl enable --now prometheus-node-exporter
```

Fase 1 — Netplan: añade la sección vlan con `<IP_DB>` (ver ejemplo).

Fase 2 — Asegurar MariaDB y configurar acceso remoto:

1. Ejecuta el script de seguridad:
```bash
sudo mysql_secure_installation
```
Sigue las opciones interactivas para establecer contraseña root y endurecer la instalación.

2. Configura `bind-address` en MariaDB para escuchar en la IP de la VLAN:
```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```
Cambia:
```
bind-address = 127.0.0.1
```
por:
```
bind-address = <IP_DB>
```
Reinicia:
```bash
sudo systemctl restart mariadb
```

3. Crear base de datos y usuarios (ajustado a MariaDB/MySQL):
```bash
sudo mysql -u root -p
```
En el prompt de MariaDB:
```sql
CREATE DATABASE db_movies;
CREATE USER 'usr_movies'@'<IP_APP1>' IDENTIFIED BY 'secret';
CREATE USER 'usr_movies'@'<IP_APP2>' IDENTIFIED BY 'secret';
GRANT ALL PRIVILEGES ON db_movies.* TO 'usr_movies'@'<IP_APP1>';
GRANT ALL PRIVILEGES ON db_movies.* TO 'usr_movies'@'<IP_APP2>';
FLUSH PRIVILEGES;

USE db_movies;
CREATE TABLE movies (
    id INT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(150) NOT NULL,
    year INT,
    UNIQUE KEY unique_title (title)
);

INSERT INTO movies (title, year) VALUES
    ('Inception', 2010),('The Matrix', 1999),('Pulp Fiction', 1994),
    ('The Dark Knight', 2008),('Eternal Sunshine of the Spotless Mind', 2004),
    ('Forrest Gump', 1994),('Fight Club', 1999),('The Godfather', 1972),
    ('Interstellar', 2014),('Parasite', 2019);
EXIT;
```

Fase 3 — UFW (permitir sólo las apps a MariaDB):
```bash
sudo ufw allow ssh
sudo ufw allow 9100/tcp
sudo ufw allow from <IP_APP1> to any port 3306
sudo ufw allow from <IP_APP2> to any port 3306
sudo ufw enable
```

Fase 4 — Verificación
- Prueba conexión desde App1/App2:
```bash
mysql -h <IP_DB> -u usr_movies -p
```
- Avisa al Proxy/monitoring que estás listo.

---

## Notas finales y recomendaciones

- Sustituye todos los placeholders `<...>` por valores reales antes de aplicar cambios.
- Mantén backups de archivos críticos (netplan, nginx, prometheus.yml, configuraciones de MariaDB).
- Si la práctica quiere que los servicios sean accesibles desde tu máquina local (host), puede que necesites configurar NAT/DNAT en VirtualBox/VM host; coordina con el instructor.
- Revisa logs si algo falla:
  - `sudo journalctl -u nginx -b`
  - `sudo journalctl -u prometheus -b`
  - `sudo journalctl -u grafana-server -b`
  - `sudo journalctl -u mariadb -b`

¡Listo! La guía ha sido limpiada, corregida y estandarizada. Puedes pedirme que:
- Aplique un ejemplo con valores reales concretos para tu entorno,
- Genere archivos de configuración listos para copiar/pegar (netplan, nginx, prometheus.yml),
- O prepare instrucciones para crear una regla NAT/DNAT en VirtualBox para acceder a Prometheus/Grafana desde tu navegador local.
