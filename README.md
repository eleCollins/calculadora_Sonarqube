# Calculadora con pruebas automáticas y SonarQube

[![Calidad y seguridad](https://github.com/eleCollins/calculadora_Sonarqube/actions/workflows/ci.yml/badge.svg)](https://github.com/eleCollins/calculadora_Sonarqube/actions/workflows/ci.yml)

Calculadora en Python hecha con programación orientada a objetos. Se puede usar desde un
menú en la terminal o por medio de una API. En cada cambio, GitHub Actions revisa
automáticamente su funcionamiento, su cobertura y su seguridad, y genera un reporte de
SonarQube que se puede descargar.

## Operaciones

| Opción | Operación | Números | Ejemplo | Resultado |
|---|---|---|---|---|
| 1 | `suma` | uno o más | `2, 3` | `5` |
| 2 | `resta` | dos (primero menos segundo) | `3, 8` | `-5` |
| 3 | `multiplicacion` | uno o más | `2, 3, 0.5` | `3` |
| 4 | `division` | dos (primero entre segundo) | `9, 2` | `4.5` |
| 5 | `promedio` | uno o más | `8, 9, 10` | `9` |
| 6 | `potencia` | dos (primero elevado al segundo) | `2, 3` | `8` |
| 7 | `mayor_menor` | uno o más | `4, -1, 7` | `mayor = 7, menor = -1` |

Los datos incorrectos, la división entre cero y los resultados demasiado grandes muestran
un mensaje comprensible en lugar de cerrar el programa.

## Estructura

```
calculadora.py              Clase Calculadora y menú de la terminal
api.py                      API con FastAPI que reutiliza la clase Calculadora
herramientas/
  reporte_sonarqube.py      Descarga el análisis de SonarQube y revisa el Quality Gate
  reporte_pdf.py            Genera el reporte en PDF
pruebas/
  test_calculadora.py       Pruebas unitarias de cada operación
  test_integracion.py       Recorridos del menú y de la API
  test_reporte.py           Pruebas de las herramientas de reporte
.github/workflows/ci.yml    Workflow de GitHub Actions
sonar-project.properties    Configuración del análisis de SonarQube
pyproject.toml              Configuración de pytest, cobertura, Ruff y Bandit
requirements.txt            Bibliotecas con versiones exactas
```

## Instalación

Requiere Python 3.12 o posterior.

macOS o Linux:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

Windows, desde Símbolo del sistema:

```bat
py -3 -m venv .venv
.venv\Scripts\activate.bat
python -m pip install -r requirements.txt
```

## Uso

### Menú en la terminal

```bash
python calculadora.py
```

Elige una operación, escribe los números y decide si quieres hacer otro cálculo.

### API

```bash
python -m uvicorn api:aplicacion
```

| Ruta | Función |
|---|---|
| `GET /estado` | Confirma que la API funciona |
| `POST /calcular` | Recibe la operación y los números, y devuelve el resultado |

Ejemplo:

```bash
curl -X POST http://127.0.0.1:8000/calcular \
  -H "Content-Type: application/json" \
  -d '{"operacion": "promedio", "numeros": [8, 9, 10]}'
```

Respuesta:

```json
{"resultado": 9.0}
```

## Pruebas y revisiones locales

```bash
python -m pytest pruebas/test_calculadora.py -v   # Pruebas unitarias
python -m pytest pruebas/test_integracion.py -v   # Pruebas de integración
python -m pytest --cov --cov-report=term-missing --cov-report=xml   # Cobertura
python -m ruff check .                            # Revisión del código
python -m ruff format --check .                   # Formato
```

La cobertura mide la calculadora, la API y las herramientas de reporte; los archivos de
pruebas no se cuentan. El proyecto exige al menos **90 %** de cobertura combinada de
líneas y ramas, y genera `coverage.xml` para SonarQube.

## GitHub Actions

El workflow [`ci.yml`](.github/workflows/ci.yml) se ejecuta con cada `push` a `main`, con
cada pull request y de forma manual (`workflow_dispatch`).

| Job | Qué hace | Espera a |
|---|---|---|
| `revision` | Revisa el código y su formato con Ruff | — |
| `unitarias` | Ejecuta las pruebas unitarias y de reportes | — |
| `integracion` | Prueba el menú y la API | `unitarias` |
| `cobertura` | Ejecuta todas las pruebas y genera la cobertura | `unitarias`, `integracion` |
| `secretos` | Busca contraseñas y credenciales con Gitleaks | — |
| `seguridad` | Revisa el código Python con Bandit | — |
| `dependencias` | Busca vulnerabilidades con pip-audit | — |
| `obsoletas` | Informa qué bibliotecas tienen versiones más recientes | — |
| `zap` | Revisa la API en funcionamiento con OWASP ZAP | `integracion`, `secretos`, `seguridad`, `dependencias` |
| `sonarqube` | Analiza el código e importa la cobertura | `cobertura`, `revision`, `secretos`, `seguridad`, `dependencias` |
| `reporte` | Genera los documentos y comprueba el Quality Gate | `sonarqube` |
| `resumen` | Muestra el estado de todos los controles, aunque alguno falle | todos |

Los fallos en pruebas, cobertura, seguridad o Quality Gate hacen que la ejecución no se
apruebe. Las dependencias obsoletas solo se informan.

`sonarqube` y `reporte` solo se ejecutan en `main`. En los pull requests aparecen como
omitidos; eso es lo esperado.

## SonarQube

- Servidor: <https://sonarqube.lynvera.ai>
- Proyecto: [`team05_calculadora_elecollins`](https://sonarqube.lynvera.ai/dashboard?id=team05_calculadora_elecollins)

La clave del proyecto aparece en `sonar-project.properties` y en
`herramientas/reporte_sonarqube.py`; ambas deben coincidir (una prueba lo verifica).

El workflow necesita el secreto `SONAR_TOKEN` en *Settings → Secrets and variables →
Actions*. Debe ser un **User Token** con permisos *Browse* y *Execute Analysis* sobre el
proyecto, porque el job `reporte` también lee los resultados. El token nunca se escribe en
los archivos del repositorio.

## Reportes descargables

Cada ejecución guarda sus resultados como *artifacts* (Actions → la ejecución → sección
*Artifacts*), o desde la terminal:

```bash
gh run download <id-de-la-ejecución> --dir reportes
```

| Artifact | Contenido |
|---|---|
| `sonarqube-report-pdf-json-md` | `sonarqube.pdf` (reporte), `sonarqube.json` (datos) y `sonarqube.md` (resumen) |
| `cobertura` | `coverage.xml` y `junit.xml` |
| `zap` | Reporte de OWASP ZAP en HTML, JSON y Markdown |
| `gitleaks` | Hallazgos de Gitleaks en formato SARIF |
| `obsoletas` | Bibliotecas con versiones más recientes |

El PDF empieza con un resumen: estado del Quality Gate, cobertura del proyecto completo y
del código nuevo, issues totales y nuevos, proyecto, fecha y commit analizado.

El *código nuevo* depende de la línea base configurada en SonarQube y no siempre equivale
al último commit. **N/D** significa que no hay una medición disponible; no equivale a cero.
