# DNS Docker - Configuración Simplificada

## Descripción
Configuración Docker con un único servidor DNS BIND9 que implementa Split Horizon DNS con logs independientes del sistema.

## Estructura del Proyecto

```
DNS-Docker-3/
├── docker-compose-new.yml    # Configuración Docker Compose
├── bind/
│   ├── named.conf            # Configuración principal BIND
│   ├── named.conf.options    # Opciones generales
│   ├── named.conf.local      # Zonas y vistas (split-horizon)
│   ├── named.conf.logging    # Configuración de logs independientes
│   ├── zones/
│   │   ├── db.internal       # Zona DNS para red interna
│   │   └── db.external       # Zona DNS para red externa
│   └── logs/                 # Logs independientes de BIND
│       ├── queries.log       # Log de consultas DNS
│       ├── general.log       # Log general
│       ├── security.log      # Log de seguridad
│       └── xfer.log          # Log de transferencias
```

## Componentes

### Contenedores
- **dnsserver**: Servidor DNS BIND9 principal
  - IP interna: 192.168.51.10
  - IP externa: 10.51.0.10
  
- **client-int**: Cliente en red interna
  - IP: 192.168.51.20
  
- **client-ext**: Cliente en red externa
  - IP: 10.51.0.20

### Redes
- **internal_net**: 192.168.51.0/24
- **external_net**: 10.51.0.0/24

## Características

### Split Horizon DNS
El servidor DNS responde de manera diferente según la red de origen:

- **Vista Interna** (192.168.51.0/24):
  - Permite recursión
  - Responde con IPs internas (192.168.51.x)
  
- **Vista Externa** (otras redes):
  - No permite recursión
  - Responde con IPs externas (10.51.0.x)

### Logs Independientes
Los logs se guardan en `./bind/logs/` y son independientes del syslog del sistema:

- **queries.log**: Todas las consultas DNS
- **general.log**: Eventos generales, configuración, red
- **security.log**: Eventos de seguridad
- **xfer.log**: Transferencias de zona y notificaciones

Cada archivo mantiene 3 versiones rotadas con tamaño máximo de 10MB.

## Uso

### 1. Detener configuración anterior (si existe)
```bash
cd ~/Documents/git_others/2018smxm7/uf1/DNS-Docker
docker compose down
```

### 2. Iniciar nueva configuración
```bash
# Renombrar el archivo docker-compose
mv docker-compose-new.yml docker-compose.yml

# Iniciar contenedores
docker compose up -d
```

### 3. Verificar estado
```bash
# Ver contenedores
docker ps -a

# Ver logs de inicio
docker logs dnsserver
```

### 4. Probar DNS desde cliente interno
```bash
# Acceder al cliente interno
docker exec -it client-int bash

# Configurar DNS
echo "nameserver 192.168.51.10" > /etc/resolv.conf

# Probar consultas
dig www.gerard.test
dig ns1.gerard.test
```

### 5. Probar DNS desde cliente externo
```bash
# Acceder al cliente externo
docker exec -it client-ext bash

# Configurar DNS
echo "nameserver 10.51.0.10" > /etc/resolv.conf

# Probar consultas
dig www.gerard.test
dig ns1.gerard.test
```

### 6. Ver logs independientes
```bash
# Ver logs de consultas
cat bind/logs/queries.log

# Ver logs generales
cat bind/logs/general.log

# Seguir logs en tiempo real
tail -f bind/logs/queries.log
```

## Verificación de Split Horizon

Desde el cliente interno deberías ver:
```
www.gerard.test.  IN  A  192.168.51.30
```

Desde el cliente externo deberías ver:
```
www.gerard.test.  IN  A  10.51.0.30
```

## Troubleshooting

### El servidor DNS no inicia
```bash
# Verificar configuración
docker exec dnsserver named-checkconf

# Verificar zonas
docker exec dnsserver named-checkzone gerard.test /etc/bind/zones/db.internal
docker exec dnsserver named-checkzone gerard.test /etc/bind/zones/db.external
```

### Ver logs detallados
```bash
docker logs dnsserver
docker exec dnsserver cat /var/cache/bind/logs/general.log
```

### Reiniciar servicio DNS
```bash
docker compose restart dnsserver
```

## Limpieza

Para detener y eliminar todo:
```bash
docker compose down
docker network prune
```
