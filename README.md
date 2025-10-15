# 🚀 Ansible Pipeline - Stack DevOps CI/CD# ansible-pipeline


## Descripción del Proyecto
Este proyecto implementa un pipeline completo de CI/CD para la aplicación web **"Teclado"** (un teclado virtual interactivo), utilizando una arquitectura distribuida en Azure con las siguientes tecnologías:

- 🔧 **Ansible**: Automatización de infraestructura y despliegues
- 🐳 **Docker & Docker Compose**: Contenedorización de servicios
- 🏗️ **Jenkins**: Servidor de integración continua
- 🔍 **SonarQube**: Análisis estático de código y calidad
- 🌐 **Nginx**: Proxy reverso para gestión de tráfico
- ☁️ **Azure**: Infraestructura en la nube (2 VMs)

---

## 🏗️ Arquitectura del Stack DevOps

### Topología

```
┌─────────────────────────────────────────┐
│  VM Nginx (4.246.162.7)                │
│  ┌────────────────────────────────┐    │
│  │  Nginx Proxy Reverso           │    │
│  │  Puerto: 80                    │    │
│  └────────────────────────────────┘    │
│         │                               │
│         │ Proxy /jenkins/ → 4.246.221.196:80
│         │ Proxy /sonar/   → 4.246.221.196:9000
└─────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│  VM Jenkins (4.246.221.196)            │
│  ┌────────────────────────────────┐    │
│  │  Docker Compose Stack          │    │
│  │  ┌──────────────────────────┐  │    │
│  │  │ Jenkins                  │  │    │
│  │  │ Puerto: 80 → 8080        │  │    │
│  │  │ Contexto: /jenkins       │  │    │
│  │  └──────────────────────────┘  │    │
│  │  ┌──────────────────────────┐  │    │
│  │  │ SonarQube                │  │    │
│  │  │ Puerto: 9000             │  │    │
│  │  └──────────────────────────┘  │    │
│  │  ┌──────────────────────────┐  │    │
│  │  │ PostgreSQL               │  │    │
│  │  │ Puerto: 5432             │  │    │
│  │  └──────────────────────────┘  │    │
│  └────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

---

## 🌐 URLs de Acceso

### A través del Proxy Nginx (Futuro - Requiere configuración)
- **Jenkins**: http://4.246.162.7/jenkins/
- **SonarQube**: http://4.246.162.7/sonar/
- **Aplicación Teclado**: http://4.246.162.7/

### Acceso Directo (Actual)
- **Jenkins**: http://4.246.221.196
- **SonarQube**: http://4.246.221.196:9000 (interno)
- **Aplicación Teclado**: http://4.246.162.7

---

## 📁 Estructura de Archivos

```
ansible-pipeline/
├── docker-compose.yml          # Stack Docker (Jenkins, SonarQube, PostgreSQL)
├── Dockerfile.jenkins          # Imagen personalizada Jenkins Alpine
├── plugins.txt                 # Plugins de Jenkins preinstalados
├── inventory.ini               # Inventario de VMs Azure
├── playbook.yml                # Playbook principal Ansible
├── requirements.txt            # Dependencias Python
├── PROBLEMAS_SOLUCIONADOS.md   # Documentación de problemas resueltos
├── templates/
│   └── nginx-base.conf.j2      # Configuración Nginx con proxy reverso
└── vars/
    └── main.yml                # Variables (IPs, puertos)
```

---

## ⚙️ Variables del Proyecto

Archivo: `vars/main.yml`

```yaml
jenkins_host: "4.246.221.196"     # IP VM Jenkins
jenkins_port: 80                   # Puerto Jenkins
sonar_host: "4.246.221.196"       # IP SonarQube
sonar_port: 9000                   # Puerto SonarQube
nginx_port: 80                     # Puerto Nginx
```

---

## 🛠️ Componentes Desplegados

### VM Jenkins (4.246.221.196)
**Contenedores Docker Compose:**

- **Jenkins**: `jenkins-custom:latest` (Alpine + JDK 17)
  - Puerto: 80 → 8080
  - Plugins: Git, SonarQube, Docker, BlueOcean, SSH Agent
  - Tools: Node.js, SonarQube Scanner 5.0.1, sshpass, rsync

- **SonarQube**: `8.2-community`
  - Puerto: 9000
  - Análisis de calidad de código

- **PostgreSQL**: `12-alpine`
  - Base de datos para SonarQube

### VM Nginx (4.246.162.7)
- Nginx nativo en Ubuntu 16.04
- Proxy reverso para Jenkins y SonarQube
- Hosting de aplicación Teclado en `/var/www/html`

---

## 📝 Problemas Resueltos

Ver documentación completa en: **[PROBLEMAS_SOLUCIONADOS.md](./PROBLEMAS_SOLUCIONADOS.md)**

1. ⚠️ **Ansible Playbook**: Error intencional (focal vs xenial repositories)
2. ⚠️ **Memoria Limitada**: Optimización con Alpine Linux (595MB vs >800MB)
3. ⚠️ **GPG Key Errors**: Cambio de Debian jdk17 a Alpine base
4. ⚠️ **Java Module System**: Flags JVM para SonarQube Scanner con Java 17
5. ⚠️ **JavaScript Analysis**: Instalación de Node.js 20.x

---

## 🚀 Pipeline CI/CD

### Flujo Automatizado

```
1. Developer → git push main
2. GitHub → Webhook → Jenkins
3. Jenkins ejecuta:
   ├─ Checkout código
   ├─ SonarQube análisis
   └─ Deploy a Nginx (SSH)
```

### Jenkinsfile

```groovy
pipeline {
  agent any
  
  environment {
    SONAR_HOST_URL = 'http://sonarqube:9000'
    SONAR_TOKEN = credentials('SONAR_TOKEN')
  }
  
  stages {
    stage('Checkout') { ... }
    stage('SonarQube analysis') { ... }
    stage('Deploy to nginx (via SSH password)') { ... }
  }
}
```

---

## 📦 Instalación y Configuración

### 1. Desplegar con Ansible

```bash
# Desde WSL Ubuntu
cd "/mnt/c/Users/Miguel Angel/Documents/Ingesoft 5/Jenkins/ansible-pipeline"

# Ejecutar playbook completo
ansible-playbook -i inventory.ini playbook.yml

# Solo Nginx
ansible-playbook -i inventory.ini playbook.yml --tags nginx

# Solo Jenkins/Docker
ansible-playbook -i inventory.ini playbook.yml --tags jenkins
```

### 2. Configurar Jenkins

```bash
# Obtener password inicial
ssh adminuser@4.246.221.196
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

**Configurar en UI:**
1. Manage Jenkins → System
2. Jenkins URL: `http://4.246.221.196`
3. SonarQube Server:
   - Name: `SonarQube`
   - URL: `http://sonarqube:9000`
   - Token: (credential `SONAR_TOKEN`)

### 3. Configurar Credenciales

#### Token SonarQube

**Generar token:**
```bash
curl -s -u admin:admin -X POST http://localhost:9000/api/user_tokens/generate -d name=jenkins-token
```

**Token obtenido**: `19bf683b399a10597b6297c8ffec712542d2168f`

**Agregar en Jenkins:**
- Manage Jenkins → Credentials → Global
- Kind: `Secret text`
- Secret: `19bf683b399a10597b6297c8ffec712542d2168f`
- ID: `SONAR_TOKEN`

#### Credenciales SSH Nginx

- Kind: `Username with password`
- Username: `adminuser`
- Password: `DevOps2025!@#`
- ID: `jenkins-pass-nginx`

### 4. Crear Pipeline Job

1. New Item → Pipeline: `Teclado-App-Pipeline`
2. Build Triggers: ✅ GitHub hook trigger
3. Pipeline:
   - Definition: `Pipeline script from SCM`
   - SCM: `Git`
   - Repository: `https://github.com/Miguel-23-ing/teclado.git`
   - Branch: `*/main`
   - Script Path: `Jenkinsfile`

---

## 🔧 Comandos Útiles

### Docker
```bash
# Ver contenedores
docker ps

# Logs
docker logs -f jenkins
docker logs -f sonarqube

# Reiniciar
docker-compose restart
```

### Nginx
```bash
# Verificar configuración
sudo nginx -t

# Recargar
sudo systemctl reload nginx

# Ver logs
sudo tail -f /var/log/nginx/access.log
```

### SonarQube
```bash
# Estado interno
curl http://localhost:9000/api/system/status

# Ver proyectos
curl -u admin:admin http://localhost:9000/api/projects/search
```

---

## 🎯 Credenciales

| Servicio | URL | Usuario | Contraseña |
|----------|-----|---------|------------|
| Jenkins | http://4.246.221.196 | admin | (ver logs iniciales) |
| SonarQube | http://4.246.221.196:9000 | admin | admin |
| VM Jenkins | SSH: 4.246.221.196 | adminuser | DevOps2025!@# |
| VM Nginx | SSH: 4.246.162.7 | adminuser | DevOps2025!@# |

---

## 🛡️ Troubleshooting

### SonarQube no responde
```bash
docker logs sonarqube | grep "SonarQube is up"
docker-compose restart sonarqube
```

### Pipeline falla en Deploy
```bash
# Probar SSH manual
sshpass -p 'DevOps2025!@#' ssh adminuser@4.246.162.7 'echo OK'

# Verificar credenciales en Jenkins
```

### Jenkins no arranca
```bash
docker logs jenkins
# Verificar memoria disponible
free -h
```

---

## 📚 Documentación Adicional

- **PROBLEMAS_SOLUCIONADOS.md**: Problemas técnicos resueltos
- **CONFIGURACION_JENKINS.md**: Guía paso a paso (en raíz del proyecto)
- **Jenkinsfile**: Pipeline completo en repositorio Teclado

---

## 👥 Equipo

**Miguel Angel** - Ingeniería de Software V

---

## 📄 Licencia

Proyecto académico - Universidad
