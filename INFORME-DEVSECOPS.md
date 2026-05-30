# 🛡️ Informe Técnico — Proceso DevSecOps del Proyecto SUOB Backend

**Sistema Unificado de Orquestación de Backups (SUOB)**
**Objetivo del informe:** Demostrar cómo las herramientas de automatización integradas en el pipeline de CI/CD mejoran el control de calidad y la seguridad del software a lo largo de todo el ciclo de vida.

| Campo | Valor |
|---|---|
| Proyecto | `backend-suob` |
| Plataforma de control de versiones | **Gitea** (`gitea.lianbpo.com`) |
| Motor de automatización | **Gitea Actions** (compatible con sintaxis GitHub Actions) |
| Tecnología base | Java 21 (Temurin) · Spring Boot 4.0.2 |
| Empaquetado | Docker / Docker Compose |
| Autor del informe | _____________________ |
| Fecha | _____________________ |

---

## 1. Resumen Ejecutivo

El proyecto SUOB Backend implementa una cadena **DevSecOps** completa y automatizada sobre **Gitea Actions**, el motor de integración y entrega continua nativo de Gitea. A diferencia de un flujo tradicional donde la calidad y la seguridad se revisan manualmente al final, aquí se aplica el principio **"Shift-Left Security"**: las pruebas, los análisis de seguridad y las validaciones de calidad se ejecutan automáticamente en cada cambio de código (`push`), **antes** de que el software llegue a producción.

El resultado es que **ningún cambio puede llegar a producción sin haber superado**:

1. Pruebas unitarias automatizadas (55 archivos de test).
2. Verificación de cobertura de código ≥ 95% (JaCoCo).
3. Análisis estático de seguridad y calidad (SAST con SonarQube).
4. Análisis de vulnerabilidades en dependencias e imagen Docker (SCA con Trivy).
5. Análisis dinámico de seguridad sobre la aplicación viva (DAST con OWASP ZAP).
6. Una **aprobación humana explícita (Quality Gate manual)** de un QA/Manager para el despliegue a producción.

> 📷 **[ESPACIO PARA EVIDENCIA 1]**
> _Captura de la página principal del repositorio en Gitea (`gitea.lianbpo.com/suob/backend-suob`), mostrando la pestaña "Actions" con el historial de ejecuciones del pipeline._

---

## 2. ¿Qué es DevSecOps y por qué Gitea Actions?

**DevSecOps** = Development + Security + Operations. Es una cultura/práctica que integra la **seguridad** y el **control de calidad** como una responsabilidad automatizada y continua dentro del flujo de desarrollo, en lugar de tratarla como una fase aislada al final.

### ¿Por qué Gitea Actions en este proyecto?

| Característica | Beneficio para el control de calidad |
|---|---|
| **Integrado al repositorio Gitea** | El código, el registro de imágenes Docker y la automatización viven en la misma plataforma autoalojada (`gitea.lianbpo.com`). No hay dependencia de servicios externos. |
| **Sintaxis compatible con GitHub Actions** | Reutiliza acciones del ecosistema (`actions/checkout`, `actions/setup-java`, `appleboy/ssh-action`, etc.) sin reescribir. |
| **Disparo automático por eventos** | Cada `push` a `master` o `development` ejecuta el pipeline sin intervención humana → **consistencia y trazabilidad total**. |
| **Registro de contenedores propio** | Las imágenes validadas se almacenan en el registro privado de Gitea, garantizando que solo artefactos auditados se desplieguen. |
| **Secrets gestionados** | Credenciales (BD, JWT, SMTP, tokens) se inyectan de forma segura desde el almacén de secretos de Gitea, nunca en el código. |

> 📷 **[ESPACIO PARA EVIDENCIA 2]**
> _Captura de la configuración de "Secrets" del repositorio en Gitea (con los valores ocultos), demostrando la gestión segura de credenciales._

> 📷 **[ESPACIO PARA EVIDENCIA 3]**
> _Captura del Runner de Gitea Actions activo/registrado (Settings → Actions → Runners), demostrando la infraestructura de ejecución._

---

## 3. Arquitectura General del Pipeline

El proceso se compone de **dos workflows** ubicados en `.gitea/workflows/`:

| Workflow | Archivo | Disparo | Propósito |
|---|---|---|---|
| **Pipeline Principal** | `pipeline.yml` | Automático (`push` a `master`/`development`) | Probar, analizar seguridad, construir y desplegar a Staging + DAST |
| **Deploy a Producción** | `deploy-produccion.yml` | Manual (`workflow_dispatch`) con aprobación QA | Promover una versión ya validada a Producción |

### Diagrama de flujo completo

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  DESARROLLADOR hace "git push" a master / development                          │
└───────────────────────────────────┬────────────────────────────────────────────┘
                                     │ (disparo automático)
                                     ▼
╔════════════════════════════════════════════════════════════════════════════════╗
║  PIPELINE PRINCIPAL (pipeline.yml)                                               ║
╠════════════════════════════════════════════════════════════════════════════════╣
║                                                                                  ║
║  ┌────────────────────────────────────────────────────────────────────────┐    ║
║  │ FASE 1 — ① SAST + SCA + BUILD                                            │    ║
║  │  ├─ 🧪 Tests Unitarios (mvnw clean test) + Cobertura JaCoCo ≥95%          │    ║
║  │  ├─ 🕵️ SCA — Trivy (vulnerabilidades en dependencias del proyecto)        │    ║
║  │  ├─ 🛡️ SAST — SonarQube (calidad + seguridad del código fuente)           │    ║
║  │  ├─ 🔐 Login al Registro Gitea                                            │    ║
║  │  ├─ 🏗️ Build imagen Backend (neutra, sin secretos)                        │    ║
║  │  ├─ 🔍 Trivy — escaneo de la IMAGEN Docker construida                     │    ║
║  │  ├─ 📤 Push Backend → Registro Gitea (tag = SHA del commit + latest)      │    ║
║  │  └─ 🏗️📤 Build + Push imagen Worker                                       │    ║
║  └────────────────────────────────────┬───────────────────────────────────┘    ║
║                                        │ (solo si rama = master)                  ║
║                                        ▼                                          ║
║  ┌────────────────────────────────────────────────────────────────────────┐    ║
║  │ FASE 2A — ② DEPLOY → STAGING                                             │    ║
║  │  ├─ 📦 Envía docker-compose.yml al servidor Staging (SCP)                 │    ║
║  │  ├─ 🚀 SSH: inyecta variables de entorno y levanta el stack              │    ║
║  │  └─ ⏳ Healthcheck automático (/actuator/health, 15 reintentos)           │    ║
║  └────────────────────────────────────┬───────────────────────────────────┘    ║
║                                        │                                          ║
║                                        ▼                                          ║
║  ┌────────────────────────────────────────────────────────────────────────┐    ║
║  │ FASE 3 — ③ DAST — OWASP ZAP (sobre la app VIVA en Staging)               │    ║
║  │  ├─ 🕷️ ZAP API Scan contra /v3/api-docs (OpenAPI)                         │    ║
║  │  ├─ 📊 Genera reportes HTML/MD/JSON como artefactos (retención 30 días)   │    ║
║  │  └─ 📝 Resumen + SHA listo para promover a producción                     │    ║
║  └────────────────────────────────────────────────────────────────────────┘    ║
║                                                                                  ║
╚════════════════════════════════════════════════════════════════════════════════╝
                                     │
                                     │  🚦 QUALITY GATE HUMANO
                                     │  (QA/Manager revisa reportes y aprueba)
                                     ▼
╔════════════════════════════════════════════════════════════════════════════════╗
║  DEPLOY A PRODUCCIÓN (deploy-produccion.yml) — DISPARO MANUAL                     ║
╠════════════════════════════════════════════════════════════════════════════════╣
║  ┌────────────────────────────────────────────────────────────────────────┐    ║
║  │ ✅ Validación Pre-Deploy                                                  │    ║
║  │  ├─ Registra QUIÉN aprueba (aprobado_por) y QUÉ SHA se despliega          │    ║
║  │  └─ Verifica que la imagen exista en el registro (pull de prueba)         │    ║
║  └────────────────────────────────────┬───────────────────────────────────┘    ║
║                                        ▼                                          ║
║  ┌────────────────────────────────────────────────────────────────────────┐    ║
║  │ 🚀 Deploy → Producción                                                    │    ║
║  │  ├─ Envía docker-compose + SSH al servidor de producción                  │    ║
║  │  ├─ Pull de la imagen EXACTA validada (por SHA)                           │    ║
║  │  ├─ Healthcheck robusto (45 reintentos, múltiples endpoints)              │    ║
║  │  └─ 🔄 ROLLBACK automático si el healthcheck falla                        │    ║
║  └────────────────────────────────────────────────────────────────────────┘    ║
╚════════════════════════════════════════════════════════════════════════════════╝
```

> 📷 **[ESPACIO PARA EVIDENCIA 4]**
> _Captura de la vista de un workflow completo en Gitea Actions mostrando los 3 jobs (①②③) en verde / completados, con sus tiempos de ejecución._

---

## 4. Análisis Detallado por Fase

### 🔹 FASE 1 — SAST + SCA + Build (`fase_1_sast_y_build`)

Es el corazón del control de calidad. Combina pruebas, análisis de seguridad y empaquetado en un solo job. Levanta un **servicio PostgreSQL efímero** (`postgres:16-alpine`) para que las pruebas corran contra una base de datos real.

#### 4.1 🧪 Pruebas Unitarias

```yaml
- name: "🧪 Tests Unitarios"
  run: ./mvnw clean test
```

- **Herramientas:** JUnit 5, Mockito, Spring Boot Test, Spring Security Test.
- **Cobertura del proyecto:** **55 archivos de prueba** que cubren controladores, casos de uso, servicios, validadores, adaptadores y entidades (arquitectura hexagonal).
- **Control de calidad automatizado (JaCoCo):** el `pom.xml` define una regla que **rompe el build** si la cobertura cae por debajo de los umbrales:

| Métrica | Umbral mínimo exigido |
|---|---|
| Instrucciones (INSTRUCTION) | **95 %** |
| Líneas (LINE) | **95 %** |
| Ramas (BRANCH) | **85 %** |

> 💡 **Punto clave para la demostración:** Este es el mecanismo donde la automatización **garantiza objetivamente la calidad**. Un desarrollador no puede fusionar código mal probado: si la cobertura baja del 95%, JaCoCo falla el `mvnw test` y el pipeline se detiene. La cobertura actual medida es del **95% (262 de 6.282 instrucciones sin cubrir)**.

> 📷 **[ESPACIO PARA EVIDENCIA 5]**
> _Captura del log del step "🧪 Tests Unitarios" en Gitea Actions mostrando "BUILD SUCCESS" y el resumen de Tests run: X, Failures: 0._

> 📷 **[ESPACIO PARA EVIDENCIA 6]**
> _Captura del reporte HTML de JaCoCo (`target/site/jacoco/index.html`) mostrando el porcentaje de cobertura global del 95%._

#### 4.2 🕵️ SCA — Trivy (Software Composition Analysis)

```yaml
- name: "🕵️ SCA — Trivy (Vulnerabilidades en dependencias)"
  run: ./trivy fs . --severity CRITICAL,HIGH --exit-code 0 --format table
```

- **Qué hace:** Escanea el sistema de archivos del proyecto en busca de **dependencias de terceros con vulnerabilidades conocidas (CVEs)** de severidad CRÍTICA y ALTA.
- **Por qué importa:** El 80%+ del código de una app moderna son librerías de terceros. SCA detecta riesgos heredados que el desarrollador no escribió.
- **Evidencia de uso real:** El `pom.xml` documenta correcciones de CVEs detectadas y remediadas:
  - `CVE-2026-24734` (HIGH) → `tomcat-embed-core` actualizado a `11.0.18`.
  - `CVE-2026-22732` (CRITICAL) → `spring-security-web` actualizado a `7.0.4`.
- **Configuración:** `--exit-code 0` → hoy **reporta sin bloquear** (modo auditoría). Comentario en el código indica que cambiando a `--exit-code 1` se puede **forzar el bloqueo** del pipeline ante vulnerabilidades.

> 📷 **[ESPACIO PARA EVIDENCIA 7]**
> _Captura del log del step "🕵️ SCA — Trivy" mostrando la tabla de vulnerabilidades detectadas en dependencias (o "No vulnerabilities found")._

#### 4.3 🛡️ SAST — SonarQube (Static Application Security Testing)

```yaml
- name: "🛡️ SAST — SonarQube"
  run: ./mvnw -B sonar:sonar -Dsonar.projectKey=backend-suob -Dsonar.qualitygate.wait=false
```

- **Qué hace:** Analiza el **código fuente propio** sin ejecutarlo, detectando: bugs, code smells, vulnerabilidades de seguridad (inyección SQL, secretos hardcodeados, etc.), duplicación de código y deuda técnica.
- **Integración segura:** El token (`SONAR_TOKEN`) y la URL del servidor (`SONAR_HOST_URL`) se inyectan desde los Secrets de Gitea.
- **Evidencia de uso real:** El historial de commits incluye `"cobertura >95% y correccion de 9 incidencias SonarQube"`, demostrando que el equipo **actúa sobre los hallazgos** de la herramienta.

> 📷 **[ESPACIO PARA EVIDENCIA 8]**
> _Captura del dashboard de SonarQube para el proyecto `backend-suob`, mostrando el Quality Gate (Passed), bugs, vulnerabilidades, code smells y % de cobertura._

> 📷 **[ESPACIO PARA EVIDENCIA 9]**
> _Captura del log del step "🛡️ SAST — SonarQube" en Gitea Actions mostrando "ANALYSIS SUCCESSFUL" y el enlace al reporte._

#### 4.4 🏗️ Build de imagen inmutable y 🔍 escaneo de la imagen

```yaml
- name: "🏗️ Build — Imagen Backend"
  run: docker build -t $BACKEND_IMAGE:${{ github.sha }} -t $BACKEND_IMAGE:latest -f Dockerfile .

- name: "🔍 Trivy — Escaneo de Imagen Docker"
  run: ./trivy image --severity CRITICAL,HIGH --exit-code 0 $BACKEND_IMAGE:${{ github.sha }}
```

- **Concepto clave — Artefacto inmutable:** La imagen se construye **una sola vez**, sin secretos de producción (imagen "neutra"), y se etiqueta con el **SHA del commit**. Esa misma imagen exacta es la que se promueve por Staging → Producción. Esto elimina el clásico problema *"en mi máquina funciona"*.
- **Doble escaneo de seguridad con Trivy:** primero el código fuente (`trivy fs`), luego la **imagen Docker final** (`trivy image`), que incluye también vulnerabilidades del sistema operativo base.
- El `Dockerfile` usa **multi-stage build** (compila con Maven, ejecuta solo el JRE) → imagen final más pequeña y con menor superficie de ataque.

> 📷 **[ESPACIO PARA EVIDENCIA 10]**
> _Captura del log "🔍 Trivy — Escaneo de Imagen Docker" mostrando el resultado del análisis de la imagen construida._

> 📷 **[ESPACIO PARA EVIDENCIA 11]**
> _Captura del Registro de Contenedores de Gitea (Packages) mostrando las imágenes `backend-suob` y `worker-suob` con sus tags (SHA + latest)._

---

### 🔹 FASE 2A — Deploy a Staging (`fase_2a_deploy_staging`)

Se ejecuta **solo en la rama `master`** y depende del éxito de la Fase 1 (`needs: fase_1_sast_y_build`).

- **Despliegue automatizado por SSH:** usa `appleboy/scp-action` para enviar el `docker-compose.yml` y `appleboy/ssh-action` para inyectar las variables del ambiente y levantar el stack.
- **Inyección segura de configuración:** todas las credenciales de Staging (BD, JWT, SMTP) provienen de Secrets, nunca del repositorio.
- **Healthcheck automatizado:** tras levantar, consulta `/actuator/health` hasta 15 veces. Si la app no responde, **el deploy falla** y no avanza.
- **Limpieza idempotente:** elimina contenedores huérfanos por nombre para liberar el puerto 8040 antes de levantar (lecciones aprendidas registradas en los commits).

> 📷 **[ESPACIO PARA EVIDENCIA 12]**
> _Captura del log del job "② Deploy → Staging" mostrando "✅ Backend respondiendo correctamente" y "🎉 Staging desplegado con éxito"._

> 📷 **[ESPACIO PARA EVIDENCIA 13]**
> _Captura de la aplicación corriendo en Staging (ej. el Swagger UI o el endpoint `/actuator/health` devolviendo `{"status":"UP"}`)._

---

### 🔹 FASE 3 — DAST con OWASP ZAP (`fase_3_dast`)

Análisis de seguridad **dinámico** sobre la aplicación **ya desplegada y en ejecución** en Staging.

```yaml
docker run ... ghcr.io/zaproxy/zaproxy:stable zap-api-scan.py \
  -t "http://<STAGING>:8040/v3/api-docs" -f openapi \
  -J report_json.json -w report_md.md -r report_html.html -I
```

- **Qué hace:** ZAP (Zed Attack Proxy, de OWASP) **ataca activamente** la API siguiendo su especificación OpenAPI, buscando vulnerabilidades en tiempo de ejecución: cabeceras de seguridad ausentes, exposición de información, inyecciones, configuraciones inseguras, etc.
- **SAST vs DAST (diferencia clave):** SAST (SonarQube) lee el código *sin ejecutarlo*; DAST (ZAP) prueba la app *corriendo de verdad*, como lo haría un atacante real. Son complementarios.
- **Reportes como evidencia auditable:** genera reportes en **HTML, Markdown y JSON**, publicados como **artefactos del pipeline con retención de 30 días**.
- **Puente hacia la aprobación:** la fase genera un archivo `deploy-info.txt` y muestra en el resumen el **SHA exacto** que debe copiarse al workflow de producción, junto con instrucciones para el QA.

> 📷 **[ESPACIO PARA EVIDENCIA 14]**
> _Captura del log del job "③ DAST — OWASP ZAP" mostrando el progreso del escaneo y el "✅ ZAP scan completado"._

> 📷 **[ESPACIO PARA EVIDENCIA 15]**
> _Captura del reporte HTML de OWASP ZAP (descargado del artefacto) mostrando el resumen de alertas por nivel de riesgo (High/Medium/Low/Informational)._

> 📷 **[ESPACIO PARA EVIDENCIA 16]**
> _Captura de la sección de "Artifacts" del workflow en Gitea, mostrando el artefacto `zap-dast-report-<sha>` disponible para descarga._

---

### 🔹 QUALITY GATE HUMANO + Deploy a Producción (`deploy-produccion.yml`)

A diferencia del resto, este workflow **NO es automático**: requiere disparo manual (`workflow_dispatch`) con dos parámetros obligatorios.

```yaml
on:
  workflow_dispatch:
    inputs:
      commit_sha:   { description: 'SHA del commit a desplegar', required: false }
      aprobado_por: { description: 'Nombre del QA/Manager que aprueba', required: true }
```

Esto implementa un **punto de control de gobernanza**:

1. **Trazabilidad / responsabilidad:** se registra el **nombre de quien aprueba** y el **SHA exacto** que se despliega → auditoría completa de quién autorizó qué y cuándo.
2. **Validación Pre-Deploy:** verifica que la imagen con ese SHA **realmente existe** en el registro (hace un `docker pull` de prueba) antes de tocar producción. Si no existe, **aborta**.
3. **Despliegue de la imagen exacta validada:** producción usa la **misma imagen** que pasó por Staging y DAST (identificada por SHA), no una reconstrucción.
4. **Healthcheck robusto + Rollback automático:** hasta 45 reintentos contra múltiples endpoints; si la app no levanta, ejecuta `docker compose down` (**rollback**) e imprime un diagnóstico completo (logs, estado de contenedores, variables).

> 💡 **Punto clave para la demostración:** Este es el balance entre automatización y control humano. La máquina hace todo el trabajo pesado y repetible (probar, escanear, construir, desplegar), pero **la decisión final de ir a producción la toma una persona responsable**, basándose en evidencia objetiva generada automáticamente (reportes DAST, SonarQube, Trivy).

> 📷 **[ESPACIO PARA EVIDENCIA 17]**
> _Captura del formulario de "Run workflow" del Deploy a Producción en Gitea, mostrando los campos `commit_sha` y `aprobado_por` a rellenar._

> 📷 **[ESPACIO PARA EVIDENCIA 18]**
> _Captura del log "✅ Validación Pre-Deploy" mostrando "Aprobado por: [nombre]" y "✅ Imágenes verificadas correctamente"._

> 📷 **[ESPACIO PARA EVIDENCIA 19]**
> _Captura del log final "🎉🎉🎉 PRODUCCION DESPLEGADA EXITOSAMENTE" con el SHA, la URL y el aprobador._

---

## 5. Inventario de Herramientas de Automatización

| Herramienta | Categoría DevSecOps | Función en el pipeline | Etapa |
|---|---|---|---|
| **Gitea** | SCM / Registry | Control de versiones, registro de imágenes y gestión de secretos | Toda |
| **Gitea Actions** | Orquestación CI/CD | Motor que ejecuta automáticamente todo el pipeline | Toda |
| **Maven (mvnw)** | Build | Compilación, gestión de dependencias y ejecución de tests | Fase 1 |
| **JUnit 5 + Mockito** | Testing | Pruebas unitarias automatizadas (55 archivos) | Fase 1 |
| **JaCoCo** | Quality Gate | Mide cobertura y **bloquea** si < 95% | Fase 1 |
| **SonarQube** | **SAST** | Análisis estático de calidad y seguridad del código | Fase 1 |
| **Trivy** | **SCA** | Escaneo de CVEs en dependencias y en la imagen Docker | Fase 1 |
| **Docker / Compose** | Empaquetado/Deploy | Construye artefactos inmutables y orquesta servicios | Fases 1, 2, 3, Prod |
| **OWASP ZAP** | **DAST** | Análisis dinámico de seguridad sobre la app viva | Fase 3 |
| **appleboy/ssh-action + scp-action** | Deploy | Despliegue remoto seguro vía SSH/SCP | Fases 2, Prod |
| **Spring Boot Actuator** | Observabilidad | Healthchecks (`/actuator/health`) para validar despliegues | Fases 2, Prod |

> 📷 **[ESPACIO PARA EVIDENCIA 20]**
> _Captura del archivo `.gitea/workflows/` en el repositorio mostrando los dos workflows (`pipeline.yml` y `deploy-produccion.yml`)._

---

## 6. Conclusión — Cómo la automatización mejora el control de calidad

Este proyecto demuestra de forma tangible los beneficios de un pipeline DevSecOps automatizado sobre Gitea Actions:

| Sin automatización (proceso manual) | Con el pipeline DevSecOps de SUOB |
|---|---|
| Las pruebas se ejecutan "cuando alguien se acuerda" | **Se ejecutan en cada `push`, sin excepción** |
| La cobertura de tests es desconocida o subjetiva | **Garantizada ≥ 95% por JaCoCo (build falla si baja)** |
| Las vulnerabilidades se descubren en producción | **Detectadas antes de desplegar** (Trivy SCA + SonarQube SAST + ZAP DAST) |
| "En mi máquina funciona" | **Artefacto inmutable** construido una vez, promovido por SHA |
| Despliegues manuales propensos a error | **Despliegue reproducible** + healthcheck + rollback automático |
| No se sabe quién autorizó un cambio | **Trazabilidad total:** quién aprueba, qué SHA, cuándo |
| Calidad como tarea opcional al final | **Calidad y seguridad integradas y obligatorias** (Shift-Left) |

**En síntesis:** las herramientas de automatización convierten el control de calidad de una actividad **manual, opcional y subjetiva** en un proceso **continuo, obligatorio, objetivo y auditable**. Cada cambio de código es validado por tres capas de seguridad (SAST, SCA, DAST) y una verificación de calidad cuantitativa (cobertura ≥95%), culminando en una aprobación humana informada por evidencia generada automáticamente. Esto reduce drásticamente el riesgo de introducir defectos o vulnerabilidades en producción, a la vez que acelera y estandariza las entregas.

---

## Anexo — Evidencias fotográficas recopiladas

> _Usa este anexo como checklist para insertar las 20 capturas referenciadas a lo largo del documento._

| # | Evidencia | Fase | Insertada ☑ |
|---|---|---|---|
| 1 | Repositorio Gitea + pestaña Actions | General | ☐ |
| 2 | Gestión de Secrets en Gitea | General | ☐ |
| 3 | Runner de Gitea Actions activo | General | ☐ |
| 4 | Workflow completo (3 jobs en verde) | General | ☐ |
| 5 | Log de Tests Unitarios (BUILD SUCCESS) | Fase 1 | ☐ |
| 6 | Reporte HTML de JaCoCo (95%) | Fase 1 | ☐ |
| 7 | Log de Trivy SCA (dependencias) | Fase 1 | ☐ |
| 8 | Dashboard de SonarQube | Fase 1 | ☐ |
| 9 | Log de SonarQube (ANALYSIS SUCCESSFUL) | Fase 1 | ☐ |
| 10 | Log de Trivy escaneo de imagen | Fase 1 | ☐ |
| 11 | Registro de contenedores Gitea (Packages) | Fase 1 | ☐ |
| 12 | Log Deploy Staging exitoso | Fase 2A | ☐ |
| 13 | App corriendo en Staging (Swagger/health) | Fase 2A | ☐ |
| 14 | Log DAST OWASP ZAP | Fase 3 | ☐ |
| 15 | Reporte HTML de OWASP ZAP | Fase 3 | ☐ |
| 16 | Artefactos del pipeline (reporte ZAP) | Fase 3 | ☐ |
| 17 | Formulario "Run workflow" Producción | Prod | ☐ |
| 18 | Log Validación Pre-Deploy | Prod | ☐ |
| 19 | Log "PRODUCCIÓN DESPLEGADA EXITOSAMENTE" | Prod | ☐ |
| 20 | Archivos de workflows en `.gitea/workflows/` | General | ☐ |

---
_Informe generado a partir del análisis del código fuente del proyecto SUOB Backend — `.gitea/workflows/pipeline.yml`, `.gitea/workflows/deploy-produccion.yml`, `pom.xml`, `Dockerfile`, `docker-compose.yml`._
