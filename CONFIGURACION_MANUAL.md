# ⚙️ Configuración Post-Instalación (Manual)

Después de ejecutar el playbook de Ansible, sigue estos pasos para configurar Jenkins y SonarQube manualmente.

---

## 📋 Pre-requisitos

✅ Playbook de Ansible ejecutado exitosamente  
✅ Contenedores Docker corriendo en la VM Jenkins (4.246.221.196)  
✅ Nginx configurado en la VM separada (4.246.162.7)  

---

## 1️⃣ Configurar Jenkins

### 1.1 Obtener Contraseña Inicial

Conéctate a la VM Jenkins y obtén la contraseña inicial:

```bash
ssh adminuser@4.246.221.196
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

**Copia la contraseña** que aparece (algo como: `a1b2c3d4e5f6...`)

### 1.2 Primer Acceso a Jenkins

1. Abre tu navegador en: **http://4.246.162.7/jenkins/** (a través del proxy Nginx)
   - O directamente: **http://4.246.221.196/jenkins/**

2. Pega la contraseña inicial que copiaste

3. Selecciona: **"Install suggested plugins"**  
   ⏳ Espera a que se instalen los plugins (2-5 minutos)

4. **Crear usuario administrador:**
   - Username: `admin`
   - Password: `DevOps2025!@#` (o la que prefieras)
   - Full name: `Administrator`
   - Email: `admin@jenkins.local`
   - Click **"Save and Continue"**

5. **Jenkins URL:**
   - Dejar como: `http://4.246.162.7/jenkins/`
   - Click **"Save and Finish"**

6. Click **"Start using Jenkins"**

---

### 1.3 Configurar SonarQube Server en Jenkins

1. En Jenkins, ve a: **Manage Jenkins** → **Configure System**

2. Busca la sección: **"SonarQube servers"**

3. Click **"Add SonarQube"**:
   - **Name:** `SonarQube`
   - **Server URL:** `http://localhost:9000/sonar/`
   - **Server authentication token:** (lo configuraremos después)

4. Por ahora, deja vacío el token y haz click en **"Save"**

---

## 2️⃣ Configurar SonarQube

### 2.1 Acceder a SonarQube

1. Abre tu navegador en: **http://4.246.162.7/sonar/** (a través del proxy)
   - O directamente: **http://4.246.221.196:9000/sonar/**

2. **Credenciales por defecto:**
   - Username: `admin`
   - Password: `admin`

3. SonarQube te pedirá cambiar la contraseña:
   - Nueva contraseña: `DevOps2025!@#` (o la que prefieras)
   - Confirmar contraseña
   - Click **"Update"**

---

### 2.2 Generar Token para Jenkins

1. En SonarQube, click en tu avatar (arriba derecha) → **"My Account"**

2. Ve a la pestaña: **"Security"**

3. En **"Generate Tokens"**:
   - **Name:** `Jenkins`
   - **Type:** `Global Analysis Token`
   - **Expires in:** `No expiration`
   - Click **"Generate"**

4. **⚠️ IMPORTANTE:** Copia el token generado (algo como: `squ_a1b2c3d4e5f6...`)
   - **¡Este token solo se muestra una vez!**
   - Guárdalo en un lugar seguro

---

### 2.3 Crear Proyecto en SonarQube

1. En el home de SonarQube, click **"Create Project"** → **"Manually"**

2. Configurar el proyecto:
   - **Project display name:** `Teclado`
   - **Project key:** `teclado`
   - **Main branch name:** `main`

3. Click **"Set Up"**

4. Click **"Locally"**

5. Selecciona: **"Use existing token"**
   - Pega el token que generaste antes (o usa el que ya tienes)

6. Click **"Continue"**

7. Selecciona: **"Other"** (para proyecto web)

8. **No ejecutes los comandos que te muestra** (Jenkins lo hará automáticamente)

---

## 3️⃣ Agregar Token de SonarQube a Jenkins

Ahora vuelve a Jenkins para agregar el token de SonarQube:

1. En Jenkins: **Manage Jenkins** → **Manage Credentials**

2. Click en **"(global)"** bajo **"Stores scoped to Jenkins"**

3. Click **"Add Credentials"** (en el menú izquierdo)

4. Configurar credencial:
   - **Kind:** `Secret text`
   - **Scope:** `Global`
   - **Secret:** (pega el token de SonarQube que generaste)
   - **ID:** `SONAR_TOKEN`
   - **Description:** `SonarQube Token`

5. Click **"Create"**

---

## 4️⃣ Actualizar Configuración de SonarQube en Jenkins

1. Ve a: **Manage Jenkins** → **Configure System**

2. Busca la sección **"SonarQube servers"**

3. En el servidor que creaste antes, ahora selecciona:
   - **Server authentication token:** `SONAR_TOKEN` (del dropdown)

4. Click **"Save"**

---

## 5️⃣ Configurar Credenciales SSH para Nginx

Para que Jenkins pueda desplegar en la VM Nginx:

1. En Jenkins: **Manage Jenkins** → **Manage Credentials**

2. Click en **"(global)"**

3. Click **"Add Credentials"**

4. Configurar:
   - **Kind:** `Username with password`
   - **Scope:** `Global`
   - **Username:** `adminuser`
   - **Password:** `DevOps2025!@#`
   - **ID:** `jenkins-pass-nginx`
   - **Description:** `SSH credentials for Nginx VM`

5. Click **"Create"**

---

## 6️⃣ Configurar GitHub Webhooks

### 6.1 En GitHub

1. Ve a tu repositorio: https://github.com/Miguel-23-ing/teclado

2. Click en **"Settings"** → **"Webhooks"** → **"Add webhook"**

3. Configurar:
   - **Payload URL:** `http://4.246.162.7/github-webhook/`
   - **Content type:** `application/json`
   - **Which events would you like to trigger this webhook?**
     - Selecciona: **"Just the push event"**
   - **Active:** ✅ (marcado)

4. Click **"Add webhook"**

5. Verifica que aparezca un ✅ verde después de unos segundos

---

## 7️⃣ Crear Pipeline Job en Jenkins

1. En Jenkins home, click **"New Item"**

2. Configurar:
   - **Enter an item name:** `teclado-pipeline`
   - Selecciona: **"Pipeline"**
   - Click **"OK"**

3. En la configuración del job:

   **General:**
   - **Description:** `Pipeline CI/CD para aplicación Teclado`

   **Build Triggers:**
   - ✅ **"GitHub hook trigger for GITScm polling"**

   **Pipeline:**
   - **Definition:** `Pipeline script from SCM`
   - **SCM:** `Git`
   - **Repository URL:** `https://github.com/Miguel-23-ing/teclado.git`
   - **Credentials:** (dejar en `-none-` si el repo es público)
   - **Branch Specifier:** `*/main`
   - **Script Path:** `Jenkinsfile`

4. Click **"Save"**

---

## 8️⃣ Ejecutar Pipeline por Primera Vez

1. En el job `teclado-pipeline`, click **"Build Now"**

2. Click en el número del build (ej: `#1`) que aparece en **"Build History"**

3. Click en **"Console Output"** para ver los logs en tiempo real

4. Verifica que todas las etapas pasen:
   - ✅ **Checkout:** Descarga código
   - ✅ **SonarQube analysis:** Analiza código
   - ✅ **Deploy to nginx:** Despliega aplicación

---

## 9️⃣ Verificar Despliegue

1. Abre tu navegador en: **http://4.246.162.7/**

2. Deberías ver la **aplicación Teclado** funcionando

3. Verifica SonarQube: **http://4.246.162.7/sonar/projects**
   - Deberías ver el proyecto `teclado` con los resultados del análisis

---

## ✅ Checklist Final

- [ ] Jenkins accesible en http://4.246.162.7/jenkins/
- [ ] SonarQube accesible en http://4.246.162.7/sonar/
- [ ] Token de SonarQube generado y agregado a Jenkins
- [ ] Credenciales SSH agregadas a Jenkins
- [ ] GitHub webhook configurado
- [ ] Pipeline `teclado-pipeline` creado
- [ ] Primer build ejecutado exitosamente
- [ ] Aplicación desplegada en http://4.246.162.7/

---

## 🚀 Flujo Automático

A partir de ahora, cada vez que hagas `git push` al repositorio:

1. GitHub enviará un webhook a Jenkins
2. Jenkins ejecutará automáticamente el pipeline
3. SonarQube analizará el código
4. Si pasa los quality gates, se desplegará automáticamente en Nginx

---

## 🔧 Troubleshooting

### Jenkins no arranca
```bash
ssh adminuser@4.246.221.196
docker logs jenkins
```

### SonarQube no accesible
```bash
docker logs sonarqube
# Esperar a que termine de inicializar (puede tomar 2-3 minutos)
```

### Error en despliegue SSH
- Verificar que las credenciales `jenkins-pass-nginx` estén correctas
- Verificar que el puerto 22 esté abierto en el NSG de Azure
- Verificar conectividad: `ping 4.246.162.7`

---

## 📊 URLs Importantes

| Servicio | URL |
|----------|-----|
| **Aplicación Teclado** | http://4.246.162.7/ |
| **Jenkins** | http://4.246.162.7/jenkins/ |
| **SonarQube** | http://4.246.162.7/sonar/ |
| **GitHub Webhook** | http://4.246.162.7/github-webhook/ |

---

¡Configuración completada! 🎉
