# TA-IND-01 — Análisis crítico de ClickHouse comparado con Elasticsearch

- **Universidad:** Universidad Técnica Estatal de Quevedo
- **Facultad:** Facultad de Ciencias de la Computación y Diseño Digital
- **Carrera:** Software
- **Asignatura:** Aplicaciones Distribuidas
- **Nivel y paralelo:** Séptimo nivel, paralelo B
- **Docente:** Gleiston Guerrero Ulloa, PhD
- **Estudiante:** Washington German Apunte Pazmiño


## Descripción

Este repositorio contiene mi informe individual TA-IND-01 de Aplicaciones Distribuidas. Analizo ClickHouse, una base de datos analítica (OLAP) de código abierto que reparte y replica sus datos entre varios nodos, y lo comparo con Elasticsearch, un motor de búsqueda y análisis distribuido que también se usa para guardar y consultar grandes volúmenes de registros (logs).

Para el experimento levanté en mi PC, con Docker, un clúster de ClickHouse con 2 réplicas y 3 nodos de ClickHouse Keeper. Detuve la copia entre réplicas, apagué una réplica, dejé a Keeper sin mayoría y comparé inserciones sin quórum y con `insert_quorum = 2` para medir cuánto cuesta la consistencia (CAP y PACELC).

El informe compilado está en [`documento/TA-IND-01_Apunte_Washington.pdf`](documento/TA-IND-01_Apunte_Washington.pdf).

## Video de sustentación

<!-- Pega aquí el enlace (YouTube no listado o Google Drive con acceso abierto). -->
Enlace: PENDIENTE

## Estructura del repositorio

| Carpeta o archivo | Contenido |
|---|---|
| `documento/` | Informe en LaTeX (`TA-IND-01_Apunte_Washington.tex`), referencias (`referencias.bib`), estilo IEEE (`ieee.bbx`, `ieee.cbx`), diagrama (`figuras/`) y PDF compilado (`TA-IND-01_Apunte_Washington.pdf`) |
| `capturas/` | Capturas de la ejecución de ClickHouse (evidencia 1) |
| `experimento/` | Comandos y resultados del experimento (evidencia 2) |
| `anexos/` | Anexo A (bosquejo a mano) y Anexo B (lecturas subrayadas) |
| `BITACORA.md` | Registro de cada sesión de trabajo |

## Cómo compilar el informe

- **Compilador:** pdfLaTeX con Biber
- **Archivo principal:** `documento/TA-IND-01_Apunte_Washington.tex`
- **Dependencias:** TeX Live o MiKTeX con los paquetes `biblatex`, `babel` (español e inglés), `lmodern`, `csquotes`, `graphicx`, `booktabs`, `tabularx`, `float`, `caption`, `xcolor`, `listings`, `appendix`, `hyperref` y `xurl`. El estilo IEEE de `biblatex` (versión 1.4g) ya viene incluido en `documento/` (`ieee.bbx` e `ieee.cbx`).
- **Importante:** se compila desde la carpeta `documento/`, porque las capturas y los anexos se cargan desde `../capturas/` y `../anexos/`.
- **Orden de comandos** (desde la carpeta `documento/`):

```
pdflatex TA-IND-01_Apunte_Washington.tex
biber TA-IND-01_Apunte_Washington
pdflatex TA-IND-01_Apunte_Washington.tex
pdflatex TA-IND-01_Apunte_Washington.tex
```

## Cómo reproducir el experimento

Requisitos: Docker Desktop y PowerShell. Desde la carpeta raíz del repositorio:

```powershell
docker compose -f experimento\cluster\docker-compose.yaml up -d
docker compose -f experimento\cluster\docker-compose.yaml ps
```

Los comandos de cada prueba están en [`experimento/README.md`](experimento/README.md), el script de la prueba de latencia en [`experimento/sql/05_latencia.sql`](experimento/sql/05_latencia.sql) y las capturas en [`capturas/`](capturas/). Las salidas guardadas de `system.query_log` están en [`experimento/resultados/`](experimento/resultados/).

Resultados que obtuve el 13/11/2026 (ClickHouse 26.10.4.207):

| Prueba | Resultado |
|---|---|
| A. Consistencia eventual | Con la copia detenida, la réplica 1 tenía 6 filas y la réplica 2 solo 1 (46 s de retraso). Al reanudarla, las dos quedaron con 6. |
| B. Caída de una réplica | Sin quórum, la escritura se aceptó; con `insert_quorum = 2`, se rechazó con el error 285 (`TOO_FEW_LIVE_REPLICAS`). |
| C. Keeper sin mayoría | La tabla quedó en solo lectura y la escritura terminó con el error 159 (`TIMEOUT_EXCEEDED`) a los 120 s. Al volver, Raft eligió como líder a `keeper-01`. |
| D. Latencia | Promedio de 18,83 ms sin quórum y de 77,23 ms con quórum de 2 (30 inserciones por modo). |

Nota: desde la versión 26.3, ClickHouse hace las inserciones de forma asíncrona por defecto (`async_insert = 1`). Por eso, en la prueba D cada inserción usa `async_insert = 0`, para que la latencia medida dependa solo del quórum.

Para apagar el clúster: `docker compose -f experimento\cluster\docker-compose.yaml stop`.
