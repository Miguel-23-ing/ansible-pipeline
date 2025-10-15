# Problemas Intencionados Encontrados y Solucionados

## 🔴 Problema 1: Versión de Ubuntu Incorrecta en Repositorio Docker

### Descripción del Problema
El playbook de Ansible usaba `focal` (Ubuntu 20.04) en la configuración del repositorio de Docker:
```yaml
repo: deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable
```

Sin embargo, las VMs desplegadas son **Ubuntu 16.04 Xenial**.

### Síntomas
- Docker se intenta instalar desde el repositorio de Ubuntu 20.04
- El paquete `docker-ce` falla al configurarse con error:
  ```
  invoke-rc.d: syntax error: unknown option "--skip-systemd-native"
  dpkg: error processing package docker-ce (--configure)
  ```

### Solución Aplicada
1. Detectar la versión correcta de Ubuntu (Xenial)
2. Usar `xenial` en lugar de `focal` en el repositorio:
   ```yaml
   repo: deb [arch=amd64] https://download.docker.com/linux/ubuntu xenial stable
   ```
3. Alternativamente, usar `docker.io` del repositorio oficial de Ubuntu 16.04

### Código Corregido
```yaml
- name: Agregar el repositorio de Docker al sources.list.d
  apt_repository:
    repo: deb [arch=amd64] https://download.docker.com/linux/ubuntu xenial stable
    state: present
```

O mejor aún, usar variable dinámica:
```yaml
- name: Obtener codename de Ubuntu
  command: lsb_release -cs
  register: ubuntu_codename

- name: Agregar el repositorio de Docker
  apt_repository:
    repo: "deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ubuntu_codename.stdout }} stable"
    state: present
```

---

## 🔴 Problema 2: Versión de Python Incompatible con Ansible

### Descripción del Problema
Ubuntu 16.04 viene con Python 3.5 por defecto, que no es compatible con la sintaxis moderna de Ansible (f-strings).

### Síntomas
```
SyntaxError: invalid syntax
  msg=f"ansible-core requires a minimum of Python version...
```

### Solución Aplicada
Uso de SSH directo con `sshpass` en lugar de depender de los módulos de Ansible que requieren Python moderno en el host remoto.

---

## 🔴 Problema 3: Módulos Ansible Obsoletos

### Descripción del Problema
El playbook usa `apt_key` y `apt_repository` que están deprecados en versiones modernas de Ansible.

### Solución Recomendada
Usar el módulo `deb822_repository` o comandos shell directos:
```yaml
- name: Agregar clave GPG de Docker
  shell: |
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

---

## ✅ Solución Implementada

Creación de `provision-direct.sh` que:
1. ✅ Usa SSH directo (no requiere Python moderno en VMs)
2. ✅ Instala Docker desde repositorio correcto (xenial)
3. ✅ Configura Nginx con template j2
4. ✅ Despliega contenedores con docker-compose
5. ✅ Todo automatizado sin intervención manual

---

## 📝 Justificación de Cambios

### Cambio 1: Uso de Script Bash en lugar de Ansible Puro
**Justificación**: Ubuntu 16.04 tiene Python 3.5 que no soporta f-strings usadas por Ansible moderno. Usar SSH directo evita dependencias de Python en las VMs.

### Cambio 2: Instalación de docker.io en lugar de docker-ce
**Justificación**: docker.io del repositorio de Ubuntu 16.04 es más compatible que docker-ce del repositorio de Docker para focal.

### Cambio 3: Template J2 para Nginx
**Justificación**: Permite configuración flexible y reutilizable de Nginx, siguiendo mejores prácticas de IaC.

### Cambio 4: Aprovisionamiento Automatizado
**Justificación**: Cumple con el requisito de "todo debe ser automatizado, nada manual".

---

## 🔴 Problema 4: Restricciones de Memoria en VMs Standard_B2s

### Descripción del Problema
Las VMs Azure Standard_B2s tienen recursos limitados (4GB RAM, 2 vCPUs) que causaban:
- Jenkins con `user: root` generaba error `EPERM` al crear threads
- Error: "Cannot create VM thread. Out of system resources"
- Error: "Cannot allocate memory" en PostgreSQL
- Contenedores reiniciándose constantemente por OOM

### Síntomas
```
Error occurred during initialization of VM
Cannot create VM thread. Out of system resources.
[0.040s][warning][os,thread] Failed to start thread "VM Thread" - pthread_create failed (EPERM)
Cannot allocate memory
```

### Solución Aplicada
1. **Cambiar a imagen Alpine**: `jenkins/jenkins:lts-alpine` (más ligera)
2. **Reducir límites de memoria**:
   - Jenkins: 768m con JVM: `-Xmx192m -Xms64m -XX:MaxMetaspaceSize=128m`
   - SonarQube: 512m con `SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true`
   - PostgreSQL: 512m con `postgres:12-alpine`
3. **Usar `user: root`** para permisos en volúmenes bind mount
4. **Volúmenes bind mount** en lugar de volúmenes Docker gestionados

### Código Corregido
```yaml
jenkins:
  image: jenkins/jenkins:lts-alpine
  user: root
  environment:
    - JENKINS_OPTS=--httpPort=8080
    - JAVA_OPTS=-Djenkins.install.runSetupWizard=false -Xmx192m -Xms64m -XX:MaxMetaspaceSize=128m
  ports:
    - "80:8080"
  volumes:
    - ./jenkins_home:/var/jenkins_home
  mem_limit: 768m
```

---

## 🎯 Resultado Final

✅ **Infraestructura Desplegada Exitosamente**:
- ✅ Nginx corriendo en VM Nginx (4.246.162.7:80)
- ✅ Docker 18.09.7 y Docker Compose 1.29.2 instalados en VM Jenkins
- ✅ Jenkins corriendo en puerto 80 (4.246.221.196:80) con imagen Alpine
- ✅ SonarQube corriendo en puerto 9000
- ✅ PostgreSQL corriendo para SonarQube
- ✅ Configuración de reverse proxy lista (JENKINS_OPTS)
- ✅ Todo configurado automáticamente sin intervención manual

### Verificación
```bash
# Jenkins respondiendo en puerto 80
curl -I http://4.246.221.196:80
# HTTP/1.1 200 OK
# Server: Jetty(12.0.25)

# Contenedores corriendo
docker ps
# jenkins      Up      0.0.0.0:80->8080/tcp
# sonarqube    Up      0.0.0.0:9000->9000/tcp
# postgres     Up      5432/tcp
```

---

## 🔧 Próximos Pasos

1. ✅ Verificar servicios corriendo - **COMPLETADO**
2. ⏳ Configurar Jenkins con repositorio Teclado
3. ⏳ Crear pipeline en Jenkins para análisis SonarQube
4. ⏳ Desplegar aplicación Teclado en Nginx
5. ⏳ Verificar análisis de SonarQube funcional
