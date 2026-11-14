# Bitácora de trabajo — TA-IND-01

Una entrada por cada día trabajado. La fecha de cada entrada debe coincidir con la de tus commits.

## Plantilla (copia este bloque para cada sesión)

### AAAA-MM-DD — Sesión N

- **Hora de inicio y fin:**
- **Qué hice:**
- **Problema que encontré:**
- **Cómo lo resolví:**
- **Tiempo dedicado:**

---

### 2026-11-12 — Sesión 1

- **Hora de inicio y fin:** 21:05 – 22:50
- **Qué hice:** Analicé el enunciado de la TA-IND-01 y los temas que ya eligieron mis compañeros. Comparé varias opciones (Milvus, ClickHouse y Bitcoin) revisando cuántos artículos con DOI hay de cada una, y elegí analizar ClickHouse y compararlo con Elasticsearch. Creé el repositorio público en GitHub, lo cloné en mi PC, agregué la estructura inicial y completé el README.
- **Problema que encontré:** Abrí PowerShell como administrador y quedé en la carpeta C:\WINDOWS\system32. También intenté editar el README directamente en GitHub, lo que habría causado un conflicto con la versión de mi PC.
- **Cómo lo resolví:** Abrí PowerShell sin permisos de administrador y me moví a mi carpeta de usuario con cd $HOME. Cancelé la edición en GitHub y modifiqué el README en VS Code antes de hacer el commit.
- **Tiempo dedicado:** 1 hora y 45 minutos aproximadamente.


---

### 2026-11-13 — Sesión 2

- **Hora de inicio y fin:** 19:23 – 22:50
- **Qué hice:** Subí la plantilla LaTeX y las carpetas del proyecto. Dibujé a mano el bosquejo de la arquitectura (Anexo A). Agregué la receta oficial del clúster de ClickHouse (1 fragmento, 2 réplicas y 3 nodos de ClickHouse Keeper) y lo levanté con Docker Compose. Tomé 11 capturas y ejecuté las cuatro pruebas: A (consistencia eventual: la réplica 2 llegó a tener 1 fila contra 6 y 46 s de atraso, y luego se igualó), B (con una réplica caída, la escritura normal se aceptó y la que pedía quórum de 2 se rechazó con el error 285), C (sin mayoría en Keeper la tabla quedó en solo lectura y, al volver, Raft eligió como nuevo líder al Keeper 1 en lugar del 3) y D (con quórum la escritura tardó 77 ms de promedio contra 19 ms sin quórum). Guardé los resultados y el registro de consultas en experimento/resultados. Preparé el PDF de una página para el SGA.
- **Problema que encontré:** En la prueba C esperaba el error TABLE_IS_READ_ONLY, pero la inserción se quedó esperando 2 minutos y terminó con TIMEOUT_EXCEEDED (159). Además, el experimento no encontraba el zip en Descargas, una captura se subió sin la otra, y la ventana de PowerShell quedó en modo "Seleccionar" y no mostraba la salida.
- **Cómo lo resolví:** Descubrí que desde la versión 26.3 ClickHouse hace las inserciones asíncronas por defecto (mi versión es la 26.10.4.207), y por eso esperó en la cola en vez de fallar al instante. Para la prueba D desactivé las inserciones asíncronas (async_insert = 0) y así medí solo el efecto del quórum. Volví a descargar el zip, subí la captura que faltaba en otro commit y salí del modo selección con Esc.
- **Tiempo dedicado:** 3 horas y 30 minutos aproximadamente.