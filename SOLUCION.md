# Solución: Plataforma CI/CD Enterprise — GlobalFin Services

Este documento contiene la propuesta técnica completa y simplificada para la plataforma de CI/CD enterprise de **GlobalFin Services** utilizando GitHub Actions, cumpliendo con los estándares de seguridad, reusabilidad, escalabilidad, mantenibilidad y optimización solicitados.

---

## Ejercicio 1 — Arquitectura pipeline enterprise

### Parte 1 — Monorepo
Para gestionar el monorepo de **GlobalFin Services**, se define una estructura ordenada y limpia donde cada componente reside en su respectivo subdirectorio:

```text
/ (Raíz del Repositorio)
├── .github/
│   └── workflows/
│       ├── main.yml                 # Orquestador principal (Selective Execution)
│       ├── reusable-test.yml        # Workflow reutilizable de Testing
│       ├── reusable-build.yml       # Workflow reutilizable de Compilación
│       └── reusable-validate.yml    # Workflow reutilizable de Validaciones Comunes
├── frontend/                        # Aplicación Frontend (React/Next.js)
├── backend/                         # Servicio API Backend (Node.js/Spring Boot)
├── infrastructure/                  # Código de Infraestructura como Código (Terraform)
├── docs/                            # Documentación del proyecto (Markdown)
└── README.md
```

### Parte 2 — Selective execution (Ejecución Selectiva)
Para evitar ejecuciones innecesarias, optimizar costes de cómputo e ignorar cambios que solo afecten a la documentación en `/docs`, utilizamos el filtro nativo de **paths** de GitHub Actions en el workflow orquestador (`main.yml`). 

#### Estrategia e Implementación:
1. **Ignorar Documentación**: Si se suben cambios únicamente en `/docs` o archivos `.md` de la raíz, el pipeline no se disparará.
2. **Ejecución dirigida**: El workflow orquestador principal detectará qué directorios han sido modificados para lanzar los jobs correspondientes de forma aislada.

Sintaxis en el orquestador (`main.yml`):
```yaml
on:
  push:
    branches:
      - main
      - staging
    paths-ignore:
      - 'docs/**'
      - '**.md'
  pull_request:
    branches:
      - main
      - staging
    paths-ignore:
      - 'docs/**'
      - '**.md'
```

A nivel de Jobs, utilizaremos una acción de filtrado de rutas certificada (`dorny/paths-filter@402f067756f71d5337ca5ad56f8f5370d9a6db6e`) con **Version Pinning** para identificar con precisión qué componente ha cambiado y definir outputs lógicos que controlen la ejecución condicional de los subsiguientes jobs.

### Parte 3 — Reusable workflows (Workflows Reutilizables)
Diseñamos tres workflows reutilizables parametrizados ubicados en `.github/workflows/`. Estos centralizan la lógica de testing, compilación y validaciones de seguridad comunes. De esta forma, cualquier nuevo servicio en el monorepo o en repositorios de la organización puede reusarlos reduciendo la duplicación de código.

#### 1. Validaciones Comunes (`reusable-validate.yml`):
Ejecuta herramientas de calidad de código y análisis estático de seguridad (SAST) de forma centralizada.
#### 2. Testing (`reusable-test.yml`):
Ejecuta el set de pruebas unitarias y de integración según el runtime y la plataforma configurada.
#### 3. Compilación (`reusable-build.yml`):
Compila y construye el artefacto tecnológico (imagen Docker, bundle, etc.) y genera la subida de un artifact seguro.

*(La sintaxis YAML exacta y completa de estos workflows reutilizables se encuentra detallada en la estructura física creada en el repositorio).*

---

## Ejercicio 2 — Seguridad enterprise

### Hardening de Permissions (Mínimo Privilegio)
Aplicamos de manera estricta el principio de mínimo privilegio. Por defecto, denegamos todos los accesos en el ámbito global del workflow y solo elevamos los permisos requeridos específicamente dentro de cada Job individual que lo necesite.

```yaml
# Configuración global restrictiva
permissions: {} # Deniega todos los permisos por defecto

jobs:
  validate:
    runs-on: ubuntu-latest
    permissions:
      contents: read   # Único permiso necesario para descargar y analizar el código
    steps:
      - uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29 # SHA Pinning (v4.1.6)
```

### Hardening de Actions Externas
1. **Version Pinning mediante SHA completo**: En lugar de utilizar tags mutables (ej. `@v4`), usamos el hash SHA commit de Git de 40 caracteres inmutable. Esto previene ataques de suplantación de tags y vulnerabilidades inyectadas en actualizaciones de terceros.
2. **Control de Supply Chain**: Solo se permite el uso de acciones oficiales de GitHub o verificadas en el Marketplace, y se auditan periódicamente sus dependencias internas.

```yaml
# Inseguro:
# uses: actions/checkout@v4

# Seguro (Enterprise Hardening):
uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29 # v4.1.6
```

### Diseño de Secrets en la Organización
Establecemos un modelo de gobernanza piramidal de secretos:
1. **Secretos de Organización**: Secretos compartidos y comunes para múltiples repositorios (ej. licencias de herramientas de escaneo, sonar tokens). Se restringen a través de políticas de repositorio permitidos.
2. **Secretos de Repositorio**: Específicos para este monorepo (ej. accesos a APIs internas compartidas).
3. **Secretos de Entorno (Environment Secrets)**: Secretos dedicados y aislados por entorno (ej. credenciales de base de datos de Staging vs Production). Solo son accesibles cuando el job se ejecuta bajo el contexto de ese entorno protegido.

### Configuración de Environments y Reglas de Protección
Configuramos dos entornos aislados:
- **`staging`**: Entorno de validación pre-producción.
- **`production`**: Entorno productivo con dos reglas de protección corporativas estrictas aplicadas desde la interfaz de GitHub:
  1. **Aprobadores Requeridos (Required Reviewers)**: Exige la firma digital aprobatoria de al menos dos ingenieros DevOps/Tech Leads antes de que el deployment comience.
  2. **Ramas de Despliegue Permitidas (Deployment Branches)**: Limita los despliegues de producción únicamente cuando provengan de la rama `main`.

```yaml
jobs:
  deploy_prod:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://api.globalfin.com
    # Este job esperará de forma nativa a la aprobación manual configurada en GitHub
```

### OIDC (OpenID Connect)
OIDC es una tecnología de federación de identidad que permite a GitHub Actions autenticarse de forma segura con proveedores Cloud (AWS, Azure, GCP) sin almacenar credenciales de larga duración en GitHub.

* **Qué problema resuelve**: Elimina la necesidad de almacenar contraseñas persistentes o Access Keys (de AWS/Azure) en los secretos de GitHub, las cuales podrían ser robadas o expirar.
* **Cómo mejora la seguridad**: GitHub genera un token OIDC (JWT) temporal de corta duración para cada ejecución. El proveedor Cloud valida que el token provenga del repositorio y workflow correcto de GitHub mediante una relación de confianza federada antes de otorgar un rol IAM temporal.
* **Cuándo utilizarlo**: Siempre que nuestros pipelines requieran interactuar con infraestructura en la nube para desplegar aplicaciones o aprovisionar recursos.

---

## Ejercicio 3 — Matrix y optimización

### Estrategia de Matrix con Include/Exclude
La estrategia de matrix nos permite paralelizar pruebas en múltiples sistemas operativos y versiones de runtime. En **GlobalFin Services** diseñamos una matriz robusta para el backend que incluye e ignora combinaciones específicas por motivos de compatibilidad empresarial:

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    runtime: [18, 20]
    include:
      # Caso especial: Añadimos un runtime heredado (legacy) pero solo sobre Linux por motivos de coste y compatibilidad
      - os: ubuntu-latest
        runtime: 16
        experimental: true
    exclude:
      # Excluimos expresamente la ejecución de Node 18 en Windows para ahorrar créditos y evitar incompatibilidad de drivers
      - os: windows-latest
        runtime: 18
```

### Técnicas de Optimización Aplicadas
1. **Cache de Dependencias**: Guardamos y restauramos dependencias (`node_modules`, `.terraform`) para evitar descargarlas en cada ejecución utilizando `actions/cache`.
2. **Paralelización**: Ejecutamos las validaciones y los tests del frontend, backend e infraestructura de manera paralela e independiente, reduciendo drásticamente el tiempo total del pipeline.
3. **Control de Concurrencia**: Usamos `concurrency` para cancelar automáticamente ejecuciones en progreso del mismo branch o Pull Request si se sube un nuevo commit, optimizando los minutos de cómputo consumidos.

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

### Reporting Dinámico en Markdown y Artifacts
Para auditoría y cumplimiento corporativo, generamos resúmenes dinámicos al finalizar el pipeline utilizando `$GITHUB_STEP_SUMMARY` y subimos archivos de reportes.

```yaml
- name: Generar Reporte Corporativo
  run: |
    echo "### :rocket: Reporte de Despliegue - GlobalFin Services" >> $GITHUB_STEP_SUMMARY
    echo "| Componente | Estado | Entorno |" >> $GITHUB_STEP_SUMMARY
    echo "|---|---|---|" >> $GITHUB_STEP_SUMMARY
    echo "| Frontend | Exitoso :white_check_mark: | Production |" >> $GITHUB_STEP_SUMMARY
    echo "| Backend | Exitoso :white_check_mark: | Production |" >> $GITHUB_STEP_SUMMARY
```

---

## Ejercicio 4 — Self-hosted runners

### Estrategia de Runners Enterprise para GlobalFin Services

#### ¿Cuándo usar Self-Hosted Runners?
- **Requisitos de Red**: Cuando el pipeline necesita acceder de forma directa a recursos dentro de una VPC privada (ej. base de datos interna, clústeres privados de Kubernetes).
- **Recursos Personalizados**: Cuando se requieren capacidades de hardware específicas (GPUs, memoria RAM alta) que no están disponibles o son muy costosas en GitHub-hosted.
- **Rendimiento y Almacenamiento**: Cuando se manejan cachés extremadamente masivas o artefactos pesados y se requiere almacenamiento de alta velocidad a nivel de disco local persistente.

#### Riesgos de Seguridad
- **Ejecución de Código no Confiable**: Si el repositorio es público o desarrolladores externos pueden ejecutar código en Pull Requests, podrían inyectar scripts maliciosos que tomen el control del servidor físico o la red privada donde corre el runner.
- **Persistencia de Estado Insegura**: A diferencia de los runners de GitHub que son efímeros e impecables en cada ejecución, los self-hosted reutilizan el sistema operativo, permitiendo que un job deje archivos temporales o credenciales expuestas para los siguientes jobs.

#### Estrategia de Mitigación y Organización
1. **Aislamiento**: Implementar runners efímeros y en contenedores (mediante herramientas como *Actions Runner Controller* en Kubernetes) que se destruyan inmediatamente después de finalizar un job.
2. **Segmentación**: Clasificar los runners por criticidad (Runners de producción en subredes aisladas y cerradas; Runners de desarrollo en subredes menos críticas).
3. **Labels**: Uso de etiquetas de hardware y entorno para dirigir los jobs con precisión.

#### Tabla Comparativa: Hosted vs Self-Hosted
| Criterio | GitHub-Hosted Runners | Self-Hosted Runners |
|---|---|---|
| **Mantenimiento** | Cero mantenimiento. GitHub se encarga de actualizaciones y parches. | Responsabilidad total del equipo DevOps interno (SO, seguridad, parches). |
| **Seguridad** | Muy alta. Entornos de máquina virtual limpios y 100% aislados por ejecución. | Requiere hardening complejo para evitar saltos de contenedor y persistencia maliciosa. |
| **Coste** | Pago por uso (minutos consumidos). | Coste fijo por infraestructura activa (servidores/cloud). |
| **Acceso Red** | Solo acceso a internet público (salvo túneles o proxies). | Acceso nativo y seguro a redes corporativas privadas y VPCs. |

#### Ejemplo Técnico de Configuración (`runs-on`)
Configuración de un job que requiere ejecutarse de manera obligatoria dentro del entorno seguro del runner corporativo:

```yaml
runs-on: [self-hosted, linux, x64, prod-network]
```

#### Estrategia de Organización
Agrupamos los runners a nivel de **Organización en GitHub** mediante **Runner Groups** dedicados. Configuramos políticas corporativas estrictas para que solo los repositorios catalogados como "Críticos de Finanzas" tengan acceso al grupo de runners de producción, impidiendo accesos accidentales desde repositorios de pruebas o sandbox.

---

## Ejercicio 5 — Troubleshooting (Resolución de Problemas)

### Caso A — El pipeline no ejecuta el deploy aunque los tests son correctos
* **Posibles Causas**:
  1. **Dependencia Faltante**: El job de deploy no tiene declarada la cláusula `needs: [test]` o el job del que depende ha fallado silenciosamente sin lanzar la señal correspondiente.
  2. **Lógica Condicional (`if`) restrictiva**: El deploy tiene una condición `if: github.ref == 'refs/heads/main'` pero la rama actual tiene un nombre ligeramente diferente (ej. `Main` o se está ejecutando desde un PR).
  3. **Aprobación de Entorno Pendiente**: El job apunta a un `environment: production` que tiene configurada la aprobación manual y está a la espera de que el aprobador corporativo autorice la ejecución.
  4. **Falta de Permisos OIDC**: El rol OIDC de la nube no está configurado para aceptar peticiones provenientes del branch específico, denegando el token de autenticación.
* **Diagnóstico**:
  - Revisar el grafo visual de GitHub Actions para verificar si el job está en estado "Skipped" (omitido), "Waiting" (esperando aprobación) o si falló la dependencia.
  - Verificar los logs iniciales del job "Set up job" para auditar las credenciales OIDC y el estado del token.
* **Solución**:
  - Corregir dependencias usando `needs` explícitamente.
  - Asegurar la sintaxis del condicional `if: github.ref == 'refs/heads/main' && success()`.
  - Asegurar que los revisores aprueben el entorno en la pestaña del pipeline de GitHub.

### Caso B — Una matrix genera más jobs de los esperados
* **Causa**: GitHub Actions realiza una multiplicación cartesiana de todos los elementos declarados en las claves principales de la matriz. Si defines `os: [ubuntu, windows]`, `runtime: [18, 20]` y `database: [postgres, mysql]`, generará `2 * 2 * 2 = 8` jobs. Si además agregas elementos al bloque `include` de forma incorrecta pensando que estás filtrando, GitHub los sumará como nuevos jobs individuales si no coinciden exactamente con las claves existentes.
* **Errores Comunes**:
  - Escribir claves no coincidentes en el bloque `include`, lo que añade nuevas filas en vez de añadir atributos a combinaciones existentes.
* **Solución**:
  - Utilizar el bloque `exclude` de manera explícita para eliminar combinaciones no deseadas.
  - Si se requiere una lista cerrada de combinaciones, en lugar de matrices multiplicativas complejas, definir las combinaciones deseadas directamente dentro de un bloque `include` vacío en la raíz de la matriz:

```yaml
# Solución elegante para un set específico de combinaciones
strategy:
  matrix:
    include:
      - os: ubuntu-latest
        runtime: 18
      - os: windows-latest
        runtime: 20
```

### Caso C — Un reusable workflow no recibe correctamente outputs
* **Posibles Causas**:
  1. **Falta de declaración en la interfaz**: El workflow reutilizable (hijo) ejecuta una acción y produce un output, pero no lo tiene expuesto formalmente bajo la directiva `on.workflow_call.outputs`.
  2. **Error de referencia de paso (scope)**: El step dentro del workflow reutilizable genera el output, pero el job que lo envuelve no lo asocia a su nivel superior (`jobs.<job_id>.outputs`).
  3. **Error al invocar en el llamador (padre)**: El workflow principal intenta acceder al output del workflow reutilizable sin declarar que depende de él mediante `needs: [reusable-job-id]`.
* **Diagnóstico**:
  - Comprobar que en el log de ejecución del paso se esté imprimiendo y escribiendo en `$GITHUB_OUTPUT` usando la sintaxis moderna: `echo "mi_output=valor" >> $GITHUB_OUTPUT`.
* **Solución**:
  - Declarar correctamente la exportación a nivel de job hijo y nivel de llamada:

```yaml
# En el workflow reutilizable (hijo)
on:
  workflow_call:
    outputs:
      build-status:
        description: "Estado de la compilación"
        value: ${{ jobs.build.outputs.status }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      status: ${{ steps.step1.outputs.my_val }}
    steps:
      - id: step1
        run: echo "my_val=success" >> $GITHUB_OUTPUT
```

---

## Preguntas teóricas cortas

### 1. Diferencia entre hosted y self-hosted runners.
Los **GitHub-hosted runners** son máquinas virtuales limpias y mantenidas 100% por GitHub que se crean bajo demanda para cada ejecución y se destruyen al finalizar. Los **Self-hosted runners** son servidores propios (físicos, virtuales o en la nube) administrados e instalados internamente por la empresa, permitiendo mayor control, hardware a medida y acceso a redes privadas, pero requiriendo mantenimiento y hardening de seguridad manual.

### 2. Diferencia entre vars y secrets.
Las **variables (`vars`)** almacenan datos de configuración no sensibles en texto plano (ej. nombres de entornos, puertos, endpoints públicos) que son legibles en los archivos de configuración y logs. Los **secretos (`secrets`)** almacenan información altamente sensible (ej. contraseñas, api keys, certificados privados) que GitHub encripta en reposo y enmascara automáticamente con asteriscos (`***`) en los logs del pipeline para evitar su exposición.

### 3. Cuándo usar reusable workflow frente a composite action.
Se utiliza un **Reusable Workflow** cuando se quiere reutilizar un flujo completo con múltiples jobs separados, aislamiento de seguridad, herencia nativa de secretos de entorno y soporte directo para ejecuciones multi-plataforma. Se utiliza una **Composite Action** cuando se quiere reutilizar una secuencia lógica de pasos (`steps`) sencillos dentro de un mismo job (compartiendo el mismo sistema de archivos y entorno de ejecución del job llamador).

### 4. Qué riesgos tiene usar actions externas sin pinning.
El mayor riesgo es el de **compromiso de la cadena de suministro (Supply Chain Attack)**. Si usamos una versión mutable como `@v1` o `@main`, un atacante que logre comprometer la cuenta del desarrollador de la acción externa podría inyectar código malicioso en esa versión. Al ejecutarse nuestro pipeline, descargaría el código comprometido y obtendría acceso de lectura a nuestros secretos, código fuente e infraestructura. El SHA pinning asegura inmutabilidad criptográfica absoluta.

### 5. Qué ventajas aporta OIDC.
OIDC (OpenID Connect) elimina las credenciales estáticas de larga duración en GitHub. Aporta un modelo de **confianza federada y credenciales de corta duración**, reduciendo la superficie de ataque drásticamente al eliminar el riesgo de robo de secretos de GitHub, automatizando además la rotación de accesos de manera nativa e invisible.

### 6. Qué contexts suelen provocar más errores.
Los contextos **`env`** y **`secrets`**. El contexto `env` no está disponible en la inicialización de ciertos parámetros del workflow (como a nivel de `concurrency` o en la propiedad `runs-on` de un job), lo que genera fallos de resolución sintáctica. El contexto `secrets` suele provocar errores de ámbito cuando se intenta pasar secretos a workflows reutilizables sin declararlos explícitamente usando la palabra clave `secrets: inherit` o asignándolos de manera directa.

### 7. Qué diferencia existe entre parse-time y runtime.
* **Parse-time (Tiempo de Análisis)**: Ocurre cuando GitHub lee y procesa la estructura sintáctica del archivo YAML antes de empezar a ejecutar cualquier job. Ejemplos de elementos evaluados en parse-time son las llaves `concurrency`, `runs-on` y directivas `on`.
* **Runtime (Tiempo de Ejecución)**: Ocurre mientras el job se está ejecutando activamente sobre el runner. Los scripts de shell dentro de `run` y el procesamiento de expresiones dinámicas basadas en variables internas como `$GITHUB_OUTPUT` ocurren en runtime.

### 8. Qué ventajas aporta concurrency.
El control de **concurrencia (`concurrency`)** garantiza que solo se ejecute un workflow o job en un grupo específico a la vez. Su principal ventaja es **prevenir conflictos de despliegue** (ej. dos despliegues simultáneos sobreescribiéndose e invalidando el estado) y **ahorrar créditos de cómputo** al cancelar automáticamente las ejecuciones obsoletas previas de una rama cuando se realiza un nuevo empuje de código.
