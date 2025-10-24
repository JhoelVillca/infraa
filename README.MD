

## 👤 Guía de Rol: PROXY + MONITOREO (Tu Rol)

**Tus IPs:**

  * **Gestión/Salto:** `192.168.100.219`
  * **Servicio (VLAN):** `<IP_PROXY>` (ej. `192.168.101.2`)

### Fase 0: Reconocimiento e Instalación

1.  **Accede a tu VM:**
      * `ssh` al "Jump Server" (ej. `adminsrv@jhoelvillca`).
      * Desde ahí, salta a tu VM: `ssh tu-usuario@192.168.100.219`
2.  **Haz reconocimiento:**
      * [cite\_start]`ip a` (Confirma el nombre de tu interfaz, probablemente `ens18`). [cite: 25]
      * `sudo ufw status` (Debería estar `inactive`).
      * `ping -c 3 8.8.8.8` (Confirma Internet).
3.  **Instala tu software:**
    ```bash
    sudo apt update
    sudo apt install nginx prometheus prometheus-node-exporter grafana -y
    sudo systemctl enable --now grafana-server prometheus-node-exporter
    ```

### Fase 1: Configuración de Red (Netplan)

1.  Haz un backup y edita el archivo Netplan:
    ```bash
    sudo cp /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.bak
    sudo nano /etc/netplan/50-cloud-init.yaml
    ```
2.  **AÑADE** la sección `vlans:` al archivo. Usa tu `<VLAN_ID>` y tu `<IP_PROXY>`.
    ```yaml
    # /etc/netplan/50-cloud-init.yaml
    network:
      version: 2
      renderer: networkd
      ethernets:
        ens18: # <-- ESTO YA EXISTE
          addresses:
          - "192.168.100.219/24"
          routes:
          - to: "default"
            via: "192.168.100.1"
          nameservers:
            addresses: [8.8.8.8]
      
      # --- ¡AÑADE ESTA PARTE! ---
      vlans:
        vlan<VLAN_ID>: # ej. vlan101
          id: <VLAN_ID> # ej. 101
          link: ens18 # ¡La interfaz que encontraste en `ip a`!
          addresses:
            - "<IP_PROXY>/29" # ej. "192.168.101.2/29"
          nameservers:
            addresses: [8.8.8.8] # ¡La clave del DNS!
          routes:
            - to: default
              via: "<IP_GATEWAY>" # ej. "192.168.101.1"
    ```
3.  Aplica y prueba:
    ```bash
    sudo netplan apply
    ping -c 3 <IP_GATEWAY> # Ping a tu Gateway de VLAN
    ```

### Fase 2: Configuración de Servicios

1.  **Nginx (Balanceador):**

    ```bash
    sudo nano /etc/nginx/sites-available/default
    ```

    *Pega esto (reemplaza `<IP_APP1>` e `<IP_APP2>`):*

    ```nginx
    upstream loadbalancer {
        server <IP_APP1>:3000; # ej. 192.168.101.3:3000
        server <IP_APP2>:3000; # ej. 192.168.101.4:3000
    }
    server {
        listen 80;
        server_name _;
        location / {
            proxy_pass http://loadbalancer;
        }
    }
    ```

    ```bash
    sudo nginx -t && sudo systemctl reload nginx
    ```

2.  **Prometheus (Colector):**

    ```bash
    sudo nano /etc/prometheus/prometheus.yml
    ```

    *Añade esto al final, bajo `scrape_configs:` (usa las IPs de *todo* tu grupo):*

    ```yaml
      - job_name: "node_exporter" # ¡El job_name correcto!
        static_configs:
          - targets:
              - "<IP_PROXY>:9100" # Tú
              - "<IP_APP1>:9100"  # App1
              - "<IP_APP2>:9100"  # App2
              - "<IP_DB>:9100"    # DB
    ```

    ```bash
    sudo systemctl restart prometheus.service
    ```

### Fase 3: Hardening (UFW)

1.  Configura tu firewall:
    ```bash
    sudo ufw allow ssh
    sudo ufw allow 80/tcp    # Nginx
    sudo ufw allow 3000/tcp  # Grafana
    sudo ufw allow 9090/tcp  # Prometheus Server
    sudo ufw allow 9100/tcp  # Node Exporter
    sudo ufw enable
    ```

### Fase 4: Verificación

1.  Abre las URLs públicas (ej. `https://vlan<VLAN_ID>-app.rootcode.com.bo`). Recarga y ve si balancea (¡dará error 502 hasta que las Apps estén listas\!).
2.  Abre Grafana (ej. `https://vlan<VLAN_ID>-monitoring.rootcode.com.bo`).
3.  Login (`admin`/`admin`), añade Data Source (Prometheus) -\> URL: `http://<IP_PROXY>:9090`.
4.  Importa Dashboard `10242`, selecciona tu Data Source, y elige `job="node_exporter"`.
5.  ¡Observa el dashboard `Targets` de Prometheus (`http://<IP_PROXY>:9090/targets` - necesitarás crear una regla DNAT y de VBox para esto si lo quieres en tu navegador) y mira cómo tus compañeros pasan de `DOWN` a `UP`\!

-----

## 🖥️ Guía de Rol: APLICACIÓN 1 (Alumno 2)

**Tus IPs:**

  * **Gestión/Salto:** `192.168.100.213`
  * **Servicio (VLAN):** `<IP_APP1>` (ej. `192.168.101.3`)

### Fase 0: Reconocimiento e Instalación

1.  Accede a tu VM: `ssh` (Jump) -\> `ssh tu-usuario@192.168.100.213`
2.  Reconoce: `ip a` (busca `ens18`), `sudo ufw status`.
3.  Instala tu software:
    ```bash
    sudo apt update
    sudo apt install prometheus-node-exporter -y
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
    \. "$HOME/.nvm/nvm.sh"
    nvm install 22
    npm install pm2@latest -g
    sudo systemctl enable --now prometheus-node-exporter
    ```

### Fase 1: Configuración de Red (Netplan)

1.  Haz backup: `sudo cp /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.bak`
2.  Edita: `sudo nano /etc/netplan/50-cloud-init.yaml`
3.  **AÑADE** tu sección de VLAN (¡usa tu IP `<IP_APP1>`\!):
    ```yaml
    # ... (Sección 'ethernets:' de gestión) ...
      vlans:
        vlan<VLAN_ID>: # ej. vlan101
          id: <VLAN_ID> # ej. 101
          link: ens18
          addresses:
            - "<IP_APP1>/29" # ej. "192.168.101.3/29"
          nameservers:
            addresses: [8.8.8.8]
          routes:
            - to: default
              via: "<IP_GATEWAY>" # ej. "192.168.101.1"
    ```
4.  Aplica y prueba: `sudo netplan apply` y `ping <IP_GATEWAY>`.

### Fase 2: Configuración de Servicios

1.  Clona y configura la App:
    ```bash
    git clone https://github.com/marceloquispeortega/api-restful-crud-movies app
    cd app
    npm install
    cp .env.example .env
    nano .env
    ```
2.  **Edita `.env`** (¡usa la IP de tu compañero de DB\!):
    ```ini
    PORT=3000
    DB_HOST='<IP_DB>' # ej. 192.168.101.5
    ... (el resto igual) ...
    ```
3.  ¡Lanza la App\!
    ```bash
    pm2 start app.js --name app1
    ```

### Fase 3: Hardening (UFW)

1.  Configura tu firewall:
    ```bash
    sudo ufw allow ssh
    sudo ufw allow 3000/tcp  # Node.js App
    sudo ufw allow 9100/tcp  # Node Exporter
    sudo ufw enable
    ```

### Fase 4: Verificación

1.  ¡Avisa al Proxy que estás en línea\!

-----

## 🖥️ Guía de Rol: APLICACIÓN 2 (Alumno 3)

**Tus IPs:**

  * **Gestión/Salto:** (La IP que te den, ej. `192.168.100.x`)
  * **Servicio (VLAN):** `<IP_APP2>` (ej. `192.168.101.4`)

**(Sigue los pasos EXACTOS de APLICACIÓN 1, pero reemplaza lo siguiente):**

  * **Fase 0 (Acceso):** Usa tu propia IP de gestión para el "salto".
  * **Fase 1 (Netplan):** Usa tu IP `<IP_APP2>` (ej. `192.168.101.4/29`).
  * **Fase 2 (PM2):** Lanza con un nombre diferente: `pm2 start app.js --name app2`

-----

## 🗄️ Guía de Rol: BASE DE DATOS (Alumno 4)

**Tus IPs:**

  * **Gestión/Salto:** `192.168.100.206`
  * **Servicio (VLAN):** `<IP_DB>` (ej. `192.168.101.5`)

### Fase 0: Reconocimiento e Instalación

1.  Accede a tu VM: `ssh` (Jump) -\> `ssh tu-usuario@192.168.100.206`
2.  Reconoce: `ip a` (busca `ens18`), `sudo ufw status`.
3.  Instala tu software:
    ```bash
    sudo apt update
    sudo apt install mariadb-server prometheus-node-exporter -y
    sudo systemctl enable --now prometheus-node-exporter
    ```

### Fase 1: Configuración de Red (Netplan)

1.  Haz backup: `sudo cp /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.bak`
2.  Edita: `sudo nano /etc/netplan/50-cloud-init.yaml`
3.  **AÑADE** tu sección de VLAN (¡usa tu IP `<IP_DB>`\!):
    ```yaml
    # ... (Sección 'ethernets:' de gestión) ...
      vlans:
        vlan<VLAN_ID>: # ej. vlan101
          id: <VLAN_ID> # ej. 101
          link: ens18
          addresses:
            - "<IP_DB>/29" # ej. "192.168.101.5/29"
          nameservers:
            addresses: [8.8.8.8]
          routes:
            - to: default
              via: "<IP_GATEWAY>" # ej. "192.168.101.1"
    ```
4.  Aplica y prueba: `sudo netplan apply` y `ping <IP_GATEWAY>`.

### Fase 2: Configuración de Servicios

1.  **Asegurar MariaDB:**

    ```bash
    sudo mysql_secure_installation
    ```

    (Presiona Enter, 'Y', pon contraseña, y 'Y' a todo).

2.  **Configurar Acceso Remoto:**

    ```bash
    sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
    ```

      * Cambia `bind-address = 127.0.0.1` por `bind-address = <IP_DB>` (ej. `192.168.101.5`).

    <!-- end list -->

    ```bash
    sudo systemctl restart mariadb
    ```

3.  **Crear BD y Usuarios:**

    ```bash
    sudo mysql -u root -p
    ```

    *Pega esto (reemplaza `<IP_APP1>` e `<IP_APP2>`):*

    ```sql
    CREATE DATABASE db_movies;
    CREATE USER 'usr_movies'@'<IP_APP1>' IDENTIFIED BY 'secret';
    CREATE USER 'usr_movies'@'<IP_APP2>' IDENTIFIED BY 'secret';
    GRANT ALL PRIVILEGES ON db_movies.* TO 'usr_movies'@'<IP_APP1>';
    GRANT ALL PRIVILEGES ON db_movies.* TO 'usr_movies'@'<IP_APP2>';
    FLUSH PRIVILEGES;
    USE db_movies;
    CREATE TABLE movies (
        id serial PRIMARY KEY,
        title character varying(150) NOT NULL,
        year integer,
        UNIQUE(title)
    );
    INSERT INTO movies (title, year) VALUES
        ('Inception', 2010),('The Matrix', 1999),('Pulp Fiction', 1994),
        ('The Dark Knight', 2008),('Eternal Sunshine of the Spotless Mind', 2004),
        ('Forrest Gump', 1994),('Fight Club', 1999),('The Godfather', 1972),
        ('Interstellar', 2014),('Parasite', 2019);
    quit;
    ```

### Fase 3: Hardening (UFW)

1.  Configura tu firewall (¡la parte más crítica\!):
    ```bash
    sudo ufw allow ssh
    sudo ufw allow 9100/tcp # Node Exporter

    # ¡PERMITIR SOLO A LAS APPS! (Reemplaza IPs)
    sudo ufw allow from <IP_APP1> to any port 3306
    sudo ufw allow from <IP_APP2> to any port 3306

    sudo ufw enable
    ```

### Fase 4: Verificación

1.  ¡Avisa al Proxy que estás en línea\!
