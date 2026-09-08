# Semana 2 · Auditoría de código con SonarQube y corrección de vulnerabilidades

**Módulo:** Desarrollo seguro, criptografía e IAM
**Semana:** 2 — Programación segura y herramientas de desarrollo seguro
**Modalidad:** laboratorio guiado, individual o en parejas
**Duración estimada:** 6 horas (2 de sesión guiada + 4 de trabajo autónomo)

---

## 1. Propósito de la actividad

En esta actividad el estudiante audita el código fuente de una aplicación web deliberadamente vulnerable (OWASP Juice Shop) usando **SonarQube**, una herramienta de análisis estático de código (SAST). A partir del reporte, selecciona tres hallazgos de seguridad, los corrige aplicando la guía de *Prácticas de Codificación Segura* de OWASP, vuelve a analizar el proyecto y demuestra la mejora. Al final realiza un escaneo dinámico corto con OWASP ZAP para comparar los dos enfoques (SAST y DAST).

Al terminar, el estudiante estará en capacidad de:

- Explicar qué es el análisis estático de código y qué tipo de problemas detecta.
- Instalar SonarQube y ejecutar un análisis sobre un proyecto real.
- Interpretar el panel de resultados: *issues*, *security hotspots*, *quality gate*.
- Corregir vulnerabilidades comunes (inyección, secretos en el código, criptografía débil) y verificar la corrección.
- Diferenciar el análisis estático (sobre el código) del análisis dinámico (sobre la aplicación en ejecución).

---

## 2. Requisitos previos

| Requisito | Detalle |
|---|---|
| Sistema operativo | **Kali Linux** (instalación nativa o máquina virtual). Si es máquina virtual: mínimo 4 vCPU, 8 GB de RAM asignados y 25 GB de disco |
| Docker | Docker Engine instalado desde los repositorios de Kali (Paso 1); es el mismo que se usó para Keycloak |
| Git | Viene preinstalado en Kali |
| Navegador | Firefox ESR (preinstalado en Kali) |
| Editor de código | Cualquiera: `mousepad` o `nano` (preinstalados) o VS Code (`sudo apt install -y code-oss`) |
| OWASP ZAP | Viene preinstalado en Kali (menú *03 - Web Application Analysis → zaproxy*). En esta guía se usa la versión en Docker para automatizar el reporte, pero puede usarse la gráfica |
| Lecturas previas | [OWASP Prácticas de Codificación Segura (español)](https://owasp.org/www-project-secure-coding-practices-quick-reference-guide/stable-es/) y [OWASP Top 10:2021 – A03 Inyección](https://owasp.org/Top10/2021/es/A03_2021-Injection/) |

> **Nota:** todos los comandos se ejecutan en la **terminal de Kali** (icono de terminal en la barra superior o `Ctrl+Alt+T`). Los comandos que empiezan por `sudo` piden la contraseña del usuario de Kali (por defecto `kali` si se usó la imagen oficial de máquina virtual). Al escribir la contraseña no se muestran caracteres; es normal.

---

## 3. Conceptos mínimos antes de empezar

- **SAST (Static Application Security Testing):** analiza el código fuente *sin ejecutarlo*, buscando patrones peligrosos (consultas SQL construidas con texto, contraseñas escritas en el código, algoritmos de cifrado obsoletos, etc.). SonarQube es una herramienta SAST.
- **DAST (Dynamic Application Security Testing):** prueba la aplicación *en ejecución*, enviándole peticiones como lo haría un atacante. OWASP ZAP es una herramienta DAST.
- **Issue:** problema detectado por SonarQube. Se clasifica por tipo (*bug*, *vulnerability*, *code smell*) y por severidad.
- **Security Hotspot:** fragmento de código sensible a la seguridad que la herramienta no puede clasificar sola como vulnerable; requiere que una persona lo revise y decida.
- **Quality Gate:** conjunto de condiciones (por ejemplo, "cero vulnerabilidades nuevas") que el proyecto debe cumplir para considerarse aprobado.

---

## 4. Paso a paso

### Paso 1. Preparar Kali Linux y Docker

1. El estudiante actualiza la lista de paquetes de Kali:

   ```bash
   sudo apt update
   ```

2. Comprueba si Docker ya está instalado (quedó instalado si se hizo el laboratorio de Keycloak):

   ```bash
   docker --version
   ```

   Si responde algo como `Docker version 27...`, salta al numeral 5. Si responde `command not found`, continúa con el numeral 3.

3. Instala Docker desde los repositorios de Kali:

   ```bash
   sudo apt install -y docker.io docker-compose
   sudo systemctl enable --now docker
   ```

4. Permite que el usuario actual use Docker sin `sudo`:

   ```bash
   sudo usermod -aG docker $USER
   ```

   **Importante:** para que el cambio aplique hay que cerrar la sesión y volver a entrar (o reiniciar la máquina virtual). Después de volver a entrar, el siguiente comando debe funcionar sin error:

   ```bash
   docker ps
   ```

   Si aparece `permission denied`, es que no se cerró la sesión; como alternativa temporal se puede ejecutar `newgrp docker` en la terminal.

5. SonarQube necesita aumentar un límite del kernel de Linux. Se aplica ahora y se deja fijo para futuros reinicios:

   ```bash
   sudo sysctl -w vm.max_map_count=262144
   echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
   ```

   Verifica que quedó aplicado (debe mostrar `262144`):

   ```bash
   sysctl vm.max_map_count
   ```

6. Crea la carpeta de trabajo y entra en ella:

   ```bash
   mkdir -p ~/semana2-sonarqube
   cd ~/semana2-sonarqube
   ```

7. Crea una red de Docker para que los contenedores del laboratorio se comuniquen entre sí por nombre:

   ```bash
   docker network create sonarnet
   ```

   Si responde que la red ya existe, no pasa nada; se puede continuar.

### Paso 2. Instalar y arrancar SonarQube

1. Descarga y arranca SonarQube Community Build (edición gratuita y de código abierto):

   ```bash
   docker run -d --name sonarqube --network sonarnet -p 9000:9000 sonarqube:community
   ```

2. Espera entre 1 y 3 minutos. Puede seguir el arranque con:

   ```bash
   docker logs -f sonarqube
   ```

   Cuando aparezca la línea `SonarQube is operational`, presiona `Ctrl+C` para salir del registro (el contenedor sigue funcionando).

3. Abre Firefox y entra a <http://localhost:9000> (o desde la terminal: `firefox http://localhost:9000 &`).

4. Inicia sesión con el usuario **admin** y la contraseña **admin**. SonarQube obligará a cambiar la contraseña: el estudiante define una nueva y la anota, la necesitará durante todo el laboratorio.

> **Evidencia 1:** captura de pantalla de la página inicial de SonarQube ya autenticado.

### Paso 3. Descargar el código de la aplicación vulnerable

1. Dentro de la carpeta `semana2-sonarqube`, clona el repositorio de OWASP Juice Shop:

   ```bash
   git clone --depth 1 https://github.com/juice-shop/juice-shop.git
   ```

2. Entra en la carpeta del proyecto y observa su estructura:

   ```bash
   cd juice-shop
   ls
   ```

   Las carpetas más importantes para esta actividad son `routes/` (lógica del servidor en TypeScript), `lib/` (utilidades) y `frontend/` (interfaz Angular).

### Paso 4. Crear el proyecto en SonarQube y obtener un token

1. En SonarQube, clic en **Create Project** (o **Projects → Create Project**) y selecciona la opción **Manually** / **Local project**.
2. Completa:
   - **Project display name:** `Juice Shop – Semana 2`
   - **Project key:** `juice-shop-semana2` (se rellena solo; no debe tener espacios)
   - **Main branch name:** `main`
3. Clic en **Next**. En la pregunta sobre la definición de código nuevo, elige **Use the global setting** y luego **Create project**.
4. En la pantalla siguiente ("How do you want to analyze your repository?") elige **Locally**.
5. Genera un token: nombre `token-semana2`, expiración *30 days*, clic en **Generate**. **Copia el token y guárdalo en un archivo de texto**; SonarQube no lo vuelve a mostrar.
6. En "Run analysis on your project" selecciona **Other (for JS, TS, Go, Python, PHP, ...)** y el sistema operativo. SonarQube muestra un comando de ejemplo; en este laboratorio se usará una variante con Docker (Paso 5), así que no es necesario instalar nada más.

### Paso 5. Configurar y ejecutar el análisis

1. Dentro de la carpeta `juice-shop`, crea un archivo llamado `sonar-project.properties`. Puede hacerlo con `nano sonar-project.properties` (para guardar: `Ctrl+O`, `Enter`, y salir con `Ctrl+X`) o con `mousepad sonar-project.properties &`. El contenido es:

   ```properties
   sonar.projectKey=juice-shop-semana2
   sonar.projectName=Juice Shop - Semana 2
   sonar.sources=.
   sonar.sourceEncoding=UTF-8
   # Se excluyen dependencias, archivos generados y datos estáticos para acelerar el análisis
   sonar.exclusions=**/node_modules/**,frontend/dist/**,frontend/src/assets/**,data/static/**,**/*.min.js,**/*.spec.ts,test/**
   ```

2. Ejecuta el escáner. Reemplaza `TOKEN_AQUI` por el token copiado en el Paso 4.

   ```bash
   docker run --rm --network sonarnet \
     -e SONAR_HOST_URL="http://sonarqube:9000" \
     -e SONAR_TOKEN="TOKEN_AQUI" \
     -v "$(pwd):/usr/src" \
     sonarsource/sonar-scanner-cli
   ```

   (Las barras `\` al final de cada línea solo indican que el comando continúa; puede escribirse todo en una sola línea sin ellas.)

3. El análisis tarda entre 5 y 15 minutos según el equipo. Termina cuando aparece `EXECUTION SUCCESS`. Si aparece `EXECUTION FAILURE`, consultar la sección 7 (Solución de problemas).

4. Vuelve al navegador, entra al proyecto en SonarQube y espera a que aparezca el panel de resultados (se refresca solo).

> **Evidencia 2:** captura del panel principal del proyecto donde se vean los contadores de *Security*, *Reliability*, *Maintainability* y el estado del *Quality Gate*.

### Paso 6. Interpretar los resultados

El estudiante explora el proyecto en SonarQube y responde en su informe:

1. ¿Cuántos *issues* de tipo **Vulnerability** y cuántos **Security Hotspots** reporta la herramienta?
2. ¿Cuál es el estado del *Quality Gate* (Passed / Failed) y qué condición lo hace fallar?
3. En la pestaña **Security Hotspots**, ¿qué categorías aparecen (por ejemplo *SQL Injection*, *Weak Cryptography*, *Authentication*)?

Para explorar:
- **Issues** → filtrar por *Type: Vulnerability* y por *Software Quality: Security*.
- **Security Hotspots** → cada hotspot muestra el fragmento de código, una explicación del riesgo ("What's the risk?") y cómo corregirlo ("How can I fix it?"). Esa explicación es la principal fuente de aprendizaje de la actividad: debe leerse con calma.

### Paso 7. Seleccionar tres hallazgos para corregir

El estudiante elige **tres hallazgos** que cumplan estas reglas:

1. **Al menos uno** debe ser de la categoría **inyección** (por ejemplo, un *hotspot* "Formatting SQL queries is security-sensitive" en `routes/login.ts` o `routes/search.ts`, donde la consulta SQL se construye concatenando texto que escribe el usuario).
2. **Al menos uno** debe ser de una categoría distinta: criptografía débil (uso de MD5/SHA-1), secretos escritos en el código, cookies inseguras, expresiones regulares vulnerables, etc.
3. Los tres deben estar en archivos diferentes.

Para cada hallazgo, el estudiante registra en una tabla: archivo y línea, regla de SonarQube (identificador tipo `typescript:S2077`), categoría, severidad, y una explicación con sus propias palabras de por qué es un riesgo. Debe relacionar cada uno con una categoría del OWASP Top 10:2021 y con el numeral correspondiente de la guía de Prácticas de Codificación Segura.

### Paso 8. Corregir el código

1. Antes de modificar nada, crea una rama de Git para los cambios:

   ```bash
   git checkout -b correcciones-semana2
   ```

2. Abre el archivo con el editor (`mousepad routes/login.ts &`, o `code-oss .` si instaló VS Code) y aplica la corrección. Orientación por tipo de hallazgo:

   - **Inyección SQL:** nunca construir la consulta pegando texto del usuario. Usar consultas parametrizadas. En Juice Shop, que utiliza Sequelize, esto significa reemplazar la concatenación por el parámetro `replacements` o por el método de consulta con objetos (`where: { email, password }`).
   - **Criptografía débil (MD5/SHA-1):** reemplazar por un algoritmo vigente. Para contraseñas, una función lenta como bcrypt o Argon2; para integridad, SHA-256 o superior.
   - **Secretos en el código:** mover el valor a una variable de entorno (`process.env.NOMBRE`) y documentar en el informe dónde debería vivir realmente.
   - **Cookies o cabeceras inseguras:** activar los atributos `httpOnly`, `secure` y `sameSite`.

   Si el estudiante no está seguro de cómo corregir, la pestaña "How can I fix it?" del hotspot muestra un ejemplo de código seguro para el mismo lenguaje.

3. Guarda los cambios y crea un *commit* por cada hallazgo corregido, con un mensaje descriptivo:

   ```bash
   git add routes/login.ts
   git commit -m "Corrige inyección SQL en login usando consulta parametrizada"
   ```

4. Genera el archivo de diferencias que se entregará como evidencia:

   ```bash
   git diff main..correcciones-semana2 > correcciones-semana2.diff
   ```

> **Evidencia 3:** el archivo `correcciones-semana2.diff`.

### Paso 9. Volver a analizar y comparar

1. Repite exactamente el comando del **Paso 5.2** (el mismo token y la misma carpeta).
2. Cuando termine, revisa en SonarQube:
   - En **Security Hotspots**, los tres hallazgos corregidos ya no deben aparecer, o deben poder marcarse como *Reviewed → Fixed*.
   - En **Activity** se ve la línea de tiempo con los dos análisis; compara los contadores.
3. Si alguno de los hallazgos sigue apareciendo, la corrección no fue suficiente: el estudiante vuelve al Paso 8 para ese caso.

> **Evidencia 4:** capturas de "antes" (Paso 5) y "después" (Paso 9) del panel del proyecto, y captura de la pestaña *Activity* donde se vean ambos análisis.

### Paso 10. Contraste con análisis dinámico (OWASP ZAP)

Esta parte muestra que hay problemas que solo se ven con la aplicación en marcha.

1. Arranca la aplicación Juice Shop original (sin corregir) en la misma red de Docker:

   ```bash
   docker run -d --name juice-shop --network sonarnet -p 3000:3000 bkimminich/juice-shop
   ```

   Comprueba en el navegador que abre en <http://localhost:3000>.

2. Ejecuta el escaneo básico de ZAP contra la aplicación (tarda 2–5 minutos):

   ```bash
   docker run --rm --network sonarnet -v "$(pwd):/zap/wrk" zaproxy/zap-stable \
     zap-baseline.py -t http://juice-shop:3000 -r reporte-zap.html
   ```

   **Alternativa con el ZAP gráfico de Kali:** abrir *Aplicaciones → 03 - Web Application Analysis → zaproxy*, elegir *Automated Scan*, escribir `http://localhost:3000` en "URL to attack", pulsar *Attack* y al terminar exportar el reporte desde *Report → Generate Report*. El resultado es equivalente; la versión en Docker se propone porque deja el archivo HTML sin pasos manuales.

3. Abre el archivo `reporte-zap.html` que quedó en la carpeta actual (`firefox reporte-zap.html &`) y responde en el informe:
   - ¿Qué alertas de nivel *Medium* o *High* reporta ZAP?
   - Elige una alerta que **no** haya aparecido en SonarQube (por ejemplo cabeceras de seguridad faltantes o configuración de cookies) y explica por qué un analizador estático no la detecta.

> **Evidencia 5:** el archivo `reporte-zap.html`.

### Paso 11. Limpiar el entorno (opcional)

Para liberar recursos al terminar (Keycloak no se ve afectado si está en otra red):

```bash
docker stop sonarqube juice-shop
docker rm sonarqube juice-shop
```

Si el estudiante prefiere conservar SonarQube para la semana siguiente, solo ejecuta `docker stop sonarqube` y más adelante `docker start sonarqube`.

---

## 5. Entregables

El estudiante sube a la tarea de Moodle (o a su repositorio de GitHub del curso) una carpeta `semana2` con:

1. **Informe** (`informe-semana2.md` o PDF, máximo 6 páginas) con:
   - Respuestas del Paso 6.
   - Tabla de los tres hallazgos (Paso 7) con su relación al OWASP Top 10 y a la guía de codificación segura.
   - Explicación de cada corrección (qué se cambió y por qué es más seguro).
   - Comparación SAST vs. DAST del Paso 10.
   - Una reflexión de un párrafo: ¿en qué fase del ciclo de vida (semana 1) debería ejecutarse cada herramienta y por qué?
2. **Evidencias 1 a 5** (capturas, `correcciones-semana2.diff`, `reporte-zap.html`).

---

## 6. Rúbrica de evaluación

| Criterio | Peso | Excelente (100 %) | Aceptable (60 %) | Insuficiente (0 %) |
|---|---|---|---|---|
| Instalación y primer análisis | 15 % | SonarQube operativo y análisis completo con evidencias | Análisis parcial o evidencias incompletas | No hay análisis |
| Interpretación de resultados | 15 % | Responde con precisión y usa la terminología correcta | Respuestas generales o con errores menores | No responde o es incorrecto |
| Selección y justificación de hallazgos | 20 % | Cumple las tres reglas; explica el riesgo con claridad y lo vincula al Top 10 | Cumple parcialmente o la explicación es superficial | Hallazgos sin justificación |
| Calidad de las correcciones | 30 % | Los tres hallazgos desaparecen en el re-análisis; el código sigue funcionando | Uno o dos corregidos, o corrección que rompe la aplicación | No hay correcciones |
| Contraste SAST/DAST | 10 % | Identifica una alerta exclusiva de ZAP y explica la diferencia | Ejecuta ZAP pero el análisis es débil | No ejecuta ZAP |
| Informe y reflexión | 10 % | Claro, ordenado, con reflexión argumentada | Desordenado o reflexión genérica | Sin informe |

---

## 7. Solución de problemas frecuentes

| Síntoma | Causa probable | Solución |
|---|---|---|
| `docker ps` responde `permission denied while trying to connect to the Docker daemon socket` | El usuario no está en el grupo `docker` o no se reinició la sesión | Ejecutar `sudo usermod -aG docker $USER`, cerrar sesión y volver a entrar (o `newgrp docker`) |
| `Cannot connect to the Docker daemon` | El servicio de Docker está detenido | `sudo systemctl start docker` |
| SonarQube se reinicia solo o los logs dicen `max virtual memory areas vm.max_map_count [65530] is too low` | Falta el ajuste del kernel | Ejecutar los comandos del Paso 1.5 y reiniciar el contenedor: `docker restart sonarqube` |
| La máquina virtual se congela durante el análisis | Poca RAM asignada | Apagar la VM y asignarle al menos 8 GB de RAM y 4 vCPU en VirtualBox/VMware; cerrar Keycloak mientras corre SonarQube (`docker stop keycloak`) |
| `http://localhost:9000` no carga | SonarQube aún está arrancando | Esperar y revisar `docker logs -f sonarqube` hasta ver `SonarQube is operational` |
| El escáner dice `Not authorized` o `401` | Token incorrecto o expirado | Generar un token nuevo en SonarQube: **My Account → Security → Generate Tokens** |
| El escáner dice `Connection refused` a `sonarqube:9000` | El contenedor no está en la red `sonarnet` | Verificar con `docker network inspect sonarnet`; si falta, recrear SonarQube con la opción `--network sonarnet` |
| El escáner se queda sin memoria (`OutOfMemoryError`) | Proyecto grande | Añadir `-e SONAR_SCANNER_OPTS="-Xmx2g"` al comando de Docker y verificar que las exclusiones del archivo `.properties` estén escritas correctamente |
| `sonar-project.properties` no se reconoce | El archivo quedó con otro nombre o en otra carpeta | Verificar con `ls -la ~/semana2-sonarqube/juice-shop/sonar-project.properties` y que el comando del escáner se ejecute desde esa carpeta |
| Puerto 9000 o 3000 ocupado | Otro servicio lo usa | Cambiar el puerto de la izquierda: `-p 9001:9000` y abrir `http://localhost:9001` |
| ZAP no llega a `juice-shop:3000` | Juice Shop no está en `sonarnet` | Recrear el contenedor con `--network sonarnet` como en el Paso 10.1 |

---

## 8. Recursos de apoyo

- [SonarQube Server – Documentación oficial (EN)](https://docs.sonarsource.com/sonarqube-server/)
- [SonarQube Community Build – Descarga (EN)](https://www.sonarsource.com/products/sonarqube/downloads/)
- [OWASP Prácticas de Codificación Segura – Guía rápida (ES)](https://owasp.org/www-project-secure-coding-practices-quick-reference-guide/stable-es/)
- [OWASP Top 10:2021 – A03 Inyección (ES)](https://owasp.org/Top10/2021/es/A03_2021-Injection/)
- [OWASP Developer Guide – Verificación: herramientas (ES)](https://devguide.owasp.org/es/06-verification/02-tools/)
- [OWASP Juice Shop – Proyecto (EN)](https://owasp.org/www-project-juice-shop/)
- [OWASP ZAP (EN)](https://www.zaproxy.org/)
- [Kali Linux – Herramienta zaproxy (EN)](https://www.kali.org/tools/zaproxy/)
- [Kali Linux – Documentación oficial (EN)](https://www.kali.org/docs/)
- [INCIBE-CERT – Fuzzing y testing (ES)](https://www.incibe.es/incibe-cert/blog/fuzzing-y-testing-sistemas-control-industrial)

---

## 9. Advertencia ética

Juice Shop es una aplicación creada para ser atacada en un entorno local. Las técnicas y herramientas de esta actividad **solo** deben usarse sobre sistemas propios o con autorización escrita. Escanear o probar aplicaciones de terceros sin permiso es ilegal en Colombia (Ley 1273 de 2009) y en la mayoría de países.

---

*Material del módulo Desarrollo seguro, criptografía e IAM · Especialización en Seguridad Informática · Licencia CC BY-SA 4.0*
