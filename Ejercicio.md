EXAMEN FINAL — GitHub Actions Enterprise GH-200
________________________________________
Duración
2 horas.
________________________________________
Escenario
La empresa “GlobalFin Services” necesita una plataforma CI/CD enterprise completamente estandarizada utilizando GitHub Actions.
La solución debe ser:
•	segura
•	reutilizable
•	escalable
•	mantenible
•	optimizada
La empresa trabaja con:
•	frontend
•	backend
•	infraestructura
•	múltiples entornos
•	políticas corporativas estrictas
________________________________________
Ejercicio 1 — Arquitectura pipeline enterprise
Objetivo
Diseñar un conjunto de workflows enterprise.
________________________________________
Requisitos
Parte 1 — Monorepo
El diseño debe contemplar:
/frontend
/backend
/infrastructure
/docs

**SOLUCIÓN — PARTE 1:**
Diseño un monorepo ordenado donde cada componente se encuentra aislado en su propia carpeta para asegurar la escalabilidad:
*   `/frontend`: Contiene el código fuente de la interfaz de usuario (React/Next.js).
*   `/backend`: Contiene los servicios API backend (Node.js/Spring Boot).
*   `/infrastructure`: Contiene el código de infraestructura como código (IaC con Terraform).
*   `/docs`: Contiene la documentación técnica del proyecto en Markdown.
*   `.github/workflows/`: Directorio centralizado donde organizo todos los archivos de configuración de GitHub Actions.

> :camera: **[CAPTURA DE PANTALLA: ESTRUCTURA DEL MONOREPO]**
> ![Estructura del Monorepo en VS Code](./img/Captura%20de%20pantalla%202026-05-29%20101121.png)
________________________________________
Parte 2 — Selective execution
Implementar lógica para:
•	Ejecutar solo pipelines necesarios
•	Ignorar documentación cuando proceda
•	Detectar componentes modificados

**SOLUCIÓN — PARTE 2:**
1.  **Ignorar Documentación**: En el disparador del orquestador principal (`main.yml`), utilizo `paths-ignore` para asegurar que cambios exclusivos en `/docs` o archivos `.md` no disparen ejecuciones de computación inútiles.
2.  **Detectar Componentes Modificados**: Utilizo la acción verificada `dorny/paths-filter` para auditar qué directorios sufrieron cambios.
3.  **Ejecutar solo pipelines necesarios**: Los resultados del filtro los exporto como `outputs` lógicos del primer job, lo cual permite que los jobs subsecuentes usen la condicional `if` para ejecutarse o saltarse según corresponda.

> :camera: **[CAPTURA DE PANTALLA: SELECTIVE EXECUTION (OMISIÓN)]**
> ![Grafo de Ejecución Selectiva - Jobs Omitidos](./img/Captura%20de%20pantalla%202026-05-29%20100727.png)

*Ejemplo en mi orquestador principal:*
```yaml
on:
  push:
    paths-ignore:
      - 'docs/**'
      - '**.md'

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      frontend: ${{ steps.filter.outputs.frontend }}
    steps:
      - uses: actions/checkout@v4
      - id: filter
        uses: dorny/paths-filter@v3
        with:
          filters: |
            frontend:
              - 'frontend/**'
```
________________________________________
Parte 3 — Reusable workflows
Crear reusable workflows para:
•	testing
•	compilación
•	validaciones comunes

**SOLUCIÓN — PARTE 3:**
He creado tres workflows reutilizables parametrizados bajo el disparador `workflow_call` en `.github/workflows/` para centralizar y estandarizar el CI/CD de la empresa:
1.  **Validaciones Comunes (`reusable-validate.yml`)**: Centraliza los análisis estáticos de código (linters) y escaneos de seguridad (SAST). Recibe el `target-path` como parámetro.
2.  **Testing (`reusable-test.yml`)**: Ejecuta el set de pruebas unitarias sobre un entorno aislado. Recibe como parámetros `node-version`, `os` y `directory`, optimizando dependencias mediante caché.
3.  **Compilación (`reusable-build.yml`)**: Compila la aplicación, genera un reporte y empaqueta el compilado usando `actions/upload-artifact`. Define outputs lógicos de éxito (`build-status`) consumibles por mi pipeline padre.

*   **Comprobación Práctica en Laboratorio (Test Realizado)**:
    Para validar estos workflows, he implementado lógica funcional y ligera en `reusable-validate.yml` que realiza un análisis sintáctico con `node --check` sobre los archivos JavaScript reales de mi monorepo (`index.js`) y un escáner de seguridad automatizado basado en `grep` para detectar posibles fugas de contraseñas u otros secretos. He verificado que los resultados de mis pruebas corren de forma exitosa en GitHub Actions.
________________________________________
Ejercicio 2 — Seguridad enterprise
Objetivo
Aplicar hardening completo.
________________________________________
Requisitos
Permissions
Aplicar:
•	permisos mínimos globales
•	permisos específicos por job

**SOLUCIÓN — PERMISSIONS:**
Aplico el principio de mínimo privilegio de manera estricta en mis diseños:
1.  **Globales**: A nivel superior de mi workflow orquestador declaro únicamente los privilegios mínimos necesarios requeridos por los workflows reutilizables y jobs generales, previniendo que cualquier paso comprometa el repositorio:
    ```yaml
    permissions:
      contents: read
      id-token: write
      pull-requests: read
    ```
2.  **Específicos**: Los jobs individuales y los workflows reutilizables definen localmente sus privilegios reducidos (por ejemplo, el job de compilación requiere únicamente `contents: read`, denegando por completo los demás ámbitos de edición).
________________________________________
Actions externas
Aplicar:
•	version pinning
•	control supply chain

**SOLUCIÓN — ACTIONS EXTERNAS:**
1.  **Version Pinning por Git Commit SHA / Tags Estables**: En mi diseño, implemento el uso de tags estables verificados o hashes SHA inmutables de 40 caracteres en lugar de tags completamente mutables (como `@main` o branches de desarrollo). Esto previene ataques de inyección en la cadena de suministro si un tag es re-apuntado maliciosamente.
    *   *Ejemplo Seguro:* `uses: actions/checkout@v4` (apuntando a una versión oficial y controlada).
2.  **Control de Supply Chain**: Recomiendo establecer políticas corporativas para admitir únicamente acciones de desarrolladores verificados en GitHub Marketplace o de repositorios internos firmados por la organización.
________________________________________
Secrets
Diseñar:
•	separación repository/org/environment
•	uso correcto scope

**SOLUCIÓN — SECRETS:**
Diseño un modelo de tres capas para la segregación segura de mis datos sensibles:
1.  **Secretos de Organización**: Para credenciales corporativas compartidas de forma controlada entre múltiples repositorios (ej. licencias globales de SonarQube).
2.  **Secretos de Repositorio**: Específicos para mi monorepo (ej. integraciones de alertas de chat, tokens internos).
3.  **Secretos de Entorno (Environment Secrets)**: Datos de alta confidencialidad (ej. credenciales de AWS de producción) que solo se cargan si el job se ejecuta bajo el contexto del entorno adecuado y tras pasar las aprobaciones correspondientes.
________________________________________
Environments
Configurar:
•	staging
•	production
Production debe:
•	requerir aprobación
•	limitar despliegues

**SOLUCIÓN — ENVIRONMENTS:**
Configuro los entornos protegidos en mi diseño de la siguiente manera:
1.  **Staging**: Entorno de validación pre-producción sin protecciones manuales estrictas.
2.  **Production**:
    *   **Aprobaciones Requeridas**: Configuro una política nativa en GitHub que exige la firma aprobatoria de al menos dos administradores antes de iniciar el despliegue.
    *   **Limitación de Despliegues**: Restrinjo los despliegues de producción para que solo puedan ejecutarse cuando los commits provengan de mi rama principal `main` (`github.ref == 'refs/heads/main'`).
________________________________________
OIDC
Explicar:
•	Qué problema resuelve
•	Cómo mejora seguridad
•	Cuándo utilizarlo

**SOLUCIÓN — OIDC:**
*   **Qué problema resuelve**: Resuelve la necesidad de almacenar credenciales Cloud de larga duración (como AWS ACCESS_KEY) en los secretos de mi repositorio de GitHub, eliminando el riesgo de robo o expiración y el esfuerzo de rotarlas manualmente.
*   **Cómo mejora la seguridad**: GitHub Actions genera dinámicamente un token OIDC (JWT) temporal de corta duración para cada ejecución. El proveedor Cloud (AWS/Azure/GCP) valida la procedencia del repositorio/workflow federado mediante una relación de confianza segura y otorga temporalmente un rol IAM de pocos minutos, reduciendo drásticamente la superficie de ataque.
*   **Cuándo utilizarlo**: Lo utilizo en todos los pipelines que requieran interactuar con plataformas en la nube para aprovisionar infraestructura o desplegar software de forma segura.
________________________________________
Ejercicio 3 — Matrix y optimización
Objetivo
Diseñar pipeline eficiente.
________________________________________
Requisitos
Matrix
Implementar:
•	múltiples plataformas
•	múltiples runtimes
•	include/exclude

**SOLUCIÓN — MATRIX:**
Para paralelizar las pruebas de mi backend en múltiples plataformas y versiones de Node.js, he configurado una matriz inteligente que excluye casos específicos por razones de coste y compatibilidad:
```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    runtime: ["18", "20"]
    exclude:
      # Exclusión corporativa para ahorrar créditos y evitar problemas en Windows
      - os: windows-latest
        runtime: "18"
```

> :camera: **[CAPTURA DE PANTALLA: EJECUCIÓN COMPLETA DE LA MATRIZ (ALL GREEN)]**
> ![Grafo de Ejecución Completa con Matriz y Test Exitosos](./img/Captura%20de%20pantalla%202026-05-29%20100948.png)
________________________________________
Optimización
Aplicar:
•	cache
•	paralelización
•	concurrency
•	reutilización lógica

**SOLUCIÓN — OPTIMIZACIÓN:**
1.  **Cache**: Utilizo la acción `actions/cache` en mis workflows reutilizables para almacenar dependencias (`node_modules`, `.npm`) basadas en el hash de los archivos lock, evitando descargas redundantes en cada ejecución.
2.  **Paralelización**: Ejecuto los jobs de validación y test de frontend, backend e infraestructura de manera paralela e independiente en diferentes runners.
3.  **Concurrency (Concurrencia)**: Agrupo mis ejecuciones dinámicamente por flujo y rama (`cancel-in-progress: true`), cancelando automáticamente ejecuciones obsoletas del mismo branch si realizo un nuevo push de código.
4.  **Reutilización de Lógica**: Centralizo las tareas en mis workflows reutilizables parametrizados.
________________________________________
Reporting
Generar:
•	summaries markdown
•	artifacts relevantes

**SOLUCIÓN — REPORTING:**
1.  **Summaries Markdown**: Al final de la ejecución, genero un reporte consolidado utilizando la variable `$GITHUB_STEP_SUMMARY` para plasmar los estados de cada job en una tabla visual.
2.  **Artifacts Relevantes**: Subo los reportes y binarios generados en la compilación usando la acción `actions/upload-artifact` a nivel de mi workflow reusable.

> :camera: **[CAPTURA DE PANTALLA: CORPORATE STEP SUMMARY REPORT]**
> ![Tabla Corporativa de Resumen de Ejecución en Markdown](./img/Captura%20de%20pantalla%202026-05-29%20101303.png)
________________________________________
Ejercicio 4 — Self-hosted runners
Objetivo
Diseñar estrategia runners enterprise.
________________________________________
Requisitos
Explicar:
•	cuándo usar self-hosted
•	riesgos seguridad
•	segmentación runners
•	labels
•	aislamiento
•	ventajas/desventajas
Debe incluirse:
•	ejemplo runs-on
•	estrategia organización runners

**SOLUCIÓN — EJERCICIO 4:**
1.  **Cuándo utilizarlos**: Los propongo cuando necesito acceso directo a recursos dentro de redes privadas internas (VPC privadas, bases de datos internas), hardware de alto rendimiento específico (GPUs, alta RAM), o control absoluto del sistema operativo del runner.
2.  **Riesgos de Seguridad**: Identifico la ejecución de código arbitrario no confiable que pueda comprometer la red corporativa, así como la persistencia de estado (los runners no son limpios por defecto y pueden conservar basura de jobs anteriores si no se configuran bien).
3.  **Mitigación y Aislamiento**: Propongo implementar runners efímeros automatizados en Kubernetes (mediante *Actions Runner Controller*) que se destruyen inmediatamente tras procesar un job, impidiendo la persistencia de datos.
4.  **Segmentación y Labels**: Etiqueto mis runners de producción con etiquetas específicas para evitar que jobs de desarrollo corran en la infraestructura productiva.

*Mi Tabla Comparativa de Ventajas y Desventajas:*
*   *Ventajas*: Acceso local a redes privadas, costes de cómputo fijos controlados, personalización de hardware a medida.
*   *Desventajas*: Asumo la responsabilidad total del mantenimiento, parches y seguridad; riesgos de secuestro de infraestructura.

*Ejemplo de mi configuración `runs-on`:*
```yaml
runs-on: [self-hosted, linux, x64, prod-network]
```

**Estrategia de Organización:**
Agrupo los runners corporativos a nivel de **Organización en GitHub** mediante **Runner Groups** seguros, aplicando políticas de accesibilidad para permitir su uso exclusivo únicamente a los repositorios que yo clasifique como críticos de producción.
________________________________________
Ejercicio 5 — Troubleshooting
Objetivo
Analizar workflows defectuosos.
________________________________________
Caso A
Un pipeline no ejecuta un deploy aunque los tests son correctos.
El alumno debe:
•	Proponer posibles causas
•	Explicar cómo diagnosticarlas
•	Explicar cómo resolverlas

**SOLUCIÓN — CASO A:**
*   **Posibles Causas que identifico**:
    1.  Falta de la cláusula de dependencia `needs: [test]` o el job previo de test falló y silenció el resultado.
    2.  Condición lógica `if` mal evaluada en mi deploy (ej. `if: github.ref == 'refs/heads/main'` pero se está ejecutando desde otra rama).
    3.  Aprobaciones pendientes bloqueando el environment en GitHub.
    4.  Falta de permisos OIDC globales denegando el token de inicio de sesión.
*   **Mi Estrategia de Diagnóstico**: Auditaría el grafo del pipeline para ver si el job está "Skipped" (omitido) u "Omitido por dependencias". Examinaría los logs iniciales de "Set up job" para revisar la validez de los permisos de tokens.
*   **Mi Propuesta de Resolución**: Configurar `needs` correctos, adecuar condicionales `if: success()` y verificar la aprobación y los branches habilitados en los ajustes del Environment de GitHub.
________________________________________
Caso B
Una matrix genera más jobs de los esperados.
El alumno debe:
•	Explicar por qué ocurre
•	Identificar errores posibles
•	Proponer solución

**SOLUCIÓN — CASO B:**
*   **Por qué ocurre**: Explico que esto ocurre debido a que GitHub Actions realiza una multiplicación cartesiana de todos los arrays provistos en los parámetros de la matriz (ej. 3 sistemas operativos * 3 runtimes = 9 jobs). Si añado objetos en el bloque `include` con llaves erróneas o de manera no coincidente, GitHub Actions los interpretará como nuevas combinaciones adicionales e incrementará la lista de jobs inesperadamente.
*   **Errores posibles que identifico**: Confundir el bloque `include` con un filtro en lugar de una adición, o cometer erratas ortográficas en los nombres de las claves del `include`.
*   **Mi Propuesta de Solución**: Sugiero utilizar el bloque `exclude` explícitamente para filtrar combinaciones. Si requiero una lista estática de combinaciones sin multiplicación cartesiana, omito las claves del nivel superior y defino la lista de combinaciones deseadas directamente dentro de un `include` vacío:
    ```yaml
    strategy:
      matrix:
        include:
          - os: ubuntu-latest
            runtime: "18"
          - os: windows-latest
            runtime: "20"
    ```
________________________________________
Caso C
Un reusable workflow no recibe correctamente outputs.
El alumno debe:
•	Explicar posibles causas
•	Identificar problemas scope
•	Explicar solución

**SOLUCIÓN — CASO C:**
*   **Posibles Causas que identifico**: El scope de los outputs en workflows reutilizables es cerrado por defecto. Los problemas comunes que identifico son:
    1.  No declarar explícitamente el output bajo `on.workflow_call.outputs` a nivel de interfaz del archivo reusable (hijo).
    2.  No mapear el output de un paso interno (`steps.mi-paso.outputs.mi-val`) con el output del job del reusable (`jobs.mi-job.outputs`).
    3.  En mi workflow llamador (padre), intentar acceder al output sin establecer la dependencia mediante `needs: [reusable-job-id]`.
*   **Mi Propuesta de Solución**: Declarar explícitamente la interfaz en ambos niveles de la siguiente forma:
    *   *En el reusable (hijo):*
        ```yaml
        on:
          workflow_call:
            outputs:
              resultado:
                value: ${{ jobs.build.outputs.status }}
        jobs:
          build:
            outputs:
              status: ${{ steps.step1.outputs.valor }}
            # ...
        ```
    *   *En el llamador (padre):* Consumo el output referenciándolo directamente con `needs.reusable-job-id.outputs.resultado`.
________________________________________
Preguntas teóricas cortas
1.	Diferencia entre hosted y self-hosted runners.
    *   **Hosted**: Servidores limpios administrados enteramente por GitHub que se crean bajo demanda para cada ejecución y se destruyen inmediatamente después de finalizar el job.
    *   **Self-hosted**: Servidores físicos o virtuales administrados por mí o mi equipo, permitiendo personalización extrema de hardware y acceso directo a redes locales privadas, pero requiriendo mantenimiento y parches de seguridad manuales.

2.	Diferencia entre vars y secrets.
    *   **Vars**: Almacenan variables y configuraciones comunes no sensibles en texto plano (ej. nombres de servidor, puertos). Son legibles en el YAML y los logs.
    *   **Secrets**: Almacenan información sensible (ej. contraseñas, api keys) que GitHub encripta en reposo y enmascara de forma automática en los logs con asteriscos (`***`) para evitar su divulgación accidental.

3.	Cuándo usar reusable workflow frente a composite action.
    *   **Reusable Workflow**: Lo utilizo para reutilizar flujos de pipeline completos con múltiples jobs, aislamiento de seguridad, herencia nativa de secretos y compatibilidad de entornos.
    *   **Composite Action**: Lo utilizo para empaquetar una secuencia básica de pasos (`steps`) reutilizables dentro de un mismo job y sistema de archivos.

4.	Qué riesgos tiene usar actions externas sin pinning.
    *   Permite ataques en la cadena de suministro si un atacante compromete la cuenta del creador de la acción y sube código malicioso sobreescribiendo el tag mutable (ej. `@v3`). Mi pipeline descargará el código malicioso automáticamente, comprometiendo variables, secretos e infraestructura.

5.	Qué ventajas aporta OIDC.
    *   Permite la federación de identidades de corta duración para conectarme a nubes (AWS/Azure/GCP) mediante tokens dinámicos temporales generados por GitHub. Resuelve el riesgo al eliminar las credenciales persistentes de larga duración en los secretos de GitHub.

6.	Qué contexts suelen provocar más errores.
    *   El contexto `env` (no disponible en la fase de parseo sintáctico de inicialización a nivel de `concurrency` o `runs-on`) y el contexto `secrets` (debido a limitaciones de scope y herencia ausente en sub-workflows reutilizables si no se declara `secrets: inherit`).

7.	Qué diferencia existe entre parse-time y runtime.
    *   **Parse-time**: Fase en la que GitHub evalúa la sintaxis estructural del YAML antes de iniciar los jobs (ej. directivas `on`, `concurrency`, `runs-on`).
    *   **Runtime**: Fase en la que el job se ejecuta activamente dentro del runner, procesando los comandos dinámicos (`run`, `$GITHUB_OUTPUT`).

8.	Qué ventajas aporta concurrency.
    *   Evita conflictos de despliegue simultáneos sobre la misma infraestructura y optimiza drásticamente el consumo de minutos de cómputo facturables al cancelar automáticamente ejecuciones obsoletas previas de una rama cuando se detecta un nuevo push.
________________________________________
Criterios de evaluación específicos
Se valorará especialmente
•	Diseño enterprise
•	Seguridad workflows
•	Capacidad troubleshooting
•	Reutilización lógica
•	Correcta explicación conceptual
•	Mantenibilidad YAML
________________________________________
ANEXO — Sintaxis de referencia permitida durante el examen
Workflow básico
name:

on:

jobs:
________________________________________
Job
job_name:
runs-on:
________________________________________
Steps
steps:
- name:
run:
________________________________________
Uses
uses: action/name@version
________________________________________
Conditions
if:
Funciones frecuentes:
success()
failure()
always()
________________________________________
Matrix
strategy:
matrix:
________________________________________
Include/exclude
include:
exclude:
________________________________________
Contexts frecuentes
github
env
vars
secrets
runner
matrix
needs
steps
________________________________________
Expressions
${{ }}
________________________________________
Outputs
GITHUB_OUTPUT
________________________________________
Reusable workflows
workflow_call:
________________________________________
Permissions
permissions:
________________________________________
Environments
environment:
________________________________________
Concurrency
concurrency:
________________________________________
Cache
key:
restore-keys:
________________________________________
Artifacts
upload-artifact
download-artifact
________________________________________
Self-hosted runners
runs-on:
Con labels:
[self-hosted, linux]
________________________________________
YAML anchors
&anchor
*anchor
<<:
________________________________________
