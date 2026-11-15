# Bitácora de trabajo — TA-IND-01

Registro de cada sesión de trabajo de la TA-IND-01.

### 2026-11-12 — Sesión 1

- **Hora de inicio y fin:** 21:05 – 22:50
- **Qué hice:** Analicé el enunciado de la TA-IND-01 y los temas que ya eligieron mis compañeros. Comparé varias opciones (Milvus, ClickHouse y Bitcoin) revisando cuántos artículos con DOI hay de cada una, y elegí analizar ClickHouse y compararlo con Elasticsearch. Creé el repositorio público en GitHub, lo cloné en mi PC, agregué la estructura inicial y completé el README.
- **Problema que encontré:** Abrí PowerShell como administrador y quedé en la carpeta C:\WINDOWS\system32. También intenté editar el README directamente en GitHub, lo que habría causado un conflicto con la versión de mi PC.
- **Cómo lo resolví:** Me moví a mi carpeta de usuario con cd $HOME, y desde ahí Git funcionó bien aunque PowerShell estuviera como administrador. Cancelé la edición en GitHub y modifiqué el README en VS Code antes de hacer el commit.
- **Tiempo dedicado:** 1 hora y 45 minutos aproximadamente.


---

### 2026-11-13 — Sesión 2

- **Hora de inicio y fin:** 19:23 – 22:50
- **Qué hice:** Subí la plantilla LaTeX y las carpetas del proyecto. Dibujé a mano el bosquejo de la arquitectura (Anexo A). Agregué la receta oficial del clúster de ClickHouse (1 fragmento, 2 réplicas y 3 nodos de ClickHouse Keeper) y lo levanté con Docker Compose. Tomé 11 capturas y ejecuté las cuatro pruebas: A (consistencia eventual: la réplica 2 llegó a tener 1 fila contra 6 y 46 s de atraso, y luego se igualó), B (con una réplica caída, la escritura normal se aceptó y la que pedía quórum de 2 se rechazó con el error 285), C (sin mayoría en Keeper la tabla quedó en solo lectura y, al volver, Raft eligió como nuevo líder al Keeper 1 en lugar del 3) y D (con quórum la escritura tardó 77 ms de promedio contra 19 ms sin quórum). Guardé los resultados y el registro de consultas en experimento/resultados. Preparé el PDF de una página para el SGA.
- **Problema que encontré:** En la prueba C esperaba el error TABLE_IS_READ_ONLY, pero la inserción se quedó esperando 2 minutos y terminó con TIMEOUT_EXCEEDED (159). Además, el experimento no encontraba el zip en Descargas, una captura se subió sin la otra, y la ventana de PowerShell quedó en modo "Seleccionar" y no mostraba la salida.
- **Cómo lo resolví:** Descubrí que desde la versión 26.3 ClickHouse hace las inserciones asíncronas por defecto (mi versión es la 26.10.4.207), y por eso esperó en la cola en vez de fallar al instante. Para la prueba D desactivé las inserciones asíncronas (async_insert = 0) y así medí solo el efecto del quórum. Volví a descargar el zip, subí la captura que faltaba en otro commit y salí del modo selección con Esc.
- **Tiempo dedicado:** 3 horas y 30 minutos aproximadamente.
---

### 2026-11-14 — Sesión 3

- **Hora de inicio y fin:** 10:54 – 13:07 y 20:02 – 22:30
- **Qué hice:** Subí las referencias en formato IEEE y el estilo de biblatex, abrí cada DOI para verificar título, autores y año, y comprobé que todos los artículos fueran de acceso abierto. Resalté y anoté tres lecturas (Schulze, Ferreira y Calderon) para el Anexo B. Escribí mis ejemplos del entorno (SGA, WhatsApp y banca móvil) y mis respuestas para cada sección; con apoyo de IA (declarado en el informe) se redactaron y corrigieron la introducción, la arquitectura, el análisis de la Unidad 1, la experimentación, la comparación con Elasticsearch, la postura crítica, las conclusiones y el resumen. Agregué el diagrama final en draw.io, renombré el archivo principal con el código de la actividad, subí el PDF compilado y completé el README. Grabé el video de sustentación y lo subí a YouTube. Al final hice una revisión completa contra la rúbrica: separé las transparencias de migración y reubicación, agregué la clasificación de sockets, RPC y mensajería, actualicé la guía del experimento con los comandos exactos que ejecuté y sus resultados, y quité comentarios de plantilla del README, de la bitácora y del .tex.
- **Problema que encontré:** El diagrama no se guardó en la carpeta documento/figuras y el primer commit no se hizo. Mi primera grabación del video duró más de 10 minutos, y el enunciado pide entre 3 y 5.
- **Cómo lo resolví:** Exporté de nuevo el diagrama, lo copié a la carpeta correcta y lo subí en otro commit. Recorté las pausas del video y lo volví a grabar más corto, siguiendo los puntos clave de cada escena.
- **Tiempo dedicado:** 4 horas y 40 minutos aproximadamente.
