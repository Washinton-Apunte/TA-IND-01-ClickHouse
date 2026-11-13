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

Para el experimento levanto en mi PC, con Docker, un clúster de ClickHouse con 2 réplicas y 3 nodos de ClickHouse Keeper. Apago una réplica para ver qué pasa con las lecturas y escrituras, y comparo inserciones con insert_quorum 1 y 2 para medir cuánto cuesta la consistencia (CAP y PACELC).

## Video de sustentación

<!-- Pega aquí el enlace (YouTube no listado o Google Drive con acceso abierto). -->
Pendiente.

## Estructura del repositorio

| Carpeta o archivo | Contenido |
|---|---|
| `documento/` | Informe en LaTeX (`main.tex`), referencias (`referencias.bib`), figuras y PDF compilado |
| `capturas/` | Capturas de la ejecución de ClickHouse (evidencia 1) |
| `experimento/` | Comandos y resultados del experimento (evidencia 2) |
| `anexos/` | Anexo A (bosquejo a mano) y Anexo B (lecturas subrayadas) |
| `BITACORA.md` | Registro de cada sesión de trabajo |

## Cómo compilar el informe

- **Compilador:** pdfLaTeX con Biber
- **Archivo principal:** `documento/main.tex`
- **Dependencias:** MiKTeX o TeX Live con los paquetes `biblatex`, `biblatex-ieee`, `babel` (español), `appendix`, `listings`, `booktabs` y `xurl`
- **Orden de comandos** (desde la carpeta `documento/`):

```
pdflatex main.tex
biber main
pdflatex main.tex
pdflatex main.tex
```

## Cómo reproducir el experimento

<!-- Se completa cuando montes el clúster de ClickHouse. -->
Pendiente.
