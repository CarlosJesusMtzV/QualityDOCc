# AGENTS.md — QualityDoc

> Reglas para cualquier agente de IA (Antigravity, Claude, etc.). **Léelo completo antes
> de escribir o modificar código.** El diseño detallado está en `docs/DISENO.md`; la
> estructura de carpetas en `docs/ESTRUCTURA.md`. No contradigas estos documentos.

## 1. Qué es
Gestión documental de calidad, **stack políglota** orquestado en Docker (gateway Nginx en :8080):
- **.NET 10 MVC** (`web`): administración, auth/RBAC, flujo de aprobación, escribe el archivo y notifica a Node.
- **Node.js/TS** (`node-search`): dueño de MongoDB; extrae metadatos en segundo plano (HTTP 202) y expone búsqueda.
- **PHP 8.3** (`php-portal`): portal de lectura (sin login); usa PostgreSQL solo para logs/reportes; consume Node.
- **Nginx**: gateway único (`/`→.NET, `/portal`→PHP, `/search`→Node, `/files`→archivos del volumen).
- **SQL Server** (núcleo) · **PostgreSQL** (auditoría) · **MongoDB** (metadatos).
- **No** hay Redis ni broker de mensajes: la integración entre lenguajes es **vía API HTTP o BD** (regla de 3 motores).

## 2. Reglas inquebrantables
1. **Patrón MVC estricto.** Controllers delgados; lógica en `Services/`; acceso a datos en `Data/`/repositorios. Nada de lógica de negocio en las vistas.
2. **Multi-tenant siempre.** Toda consulta de negocio filtra por `EmpresaId` (Global Query Filter de EF). Solo el SuperAdmin usa `IgnoreQueryFilters()`.
3. **Nunca se sobreescribe un archivo físico.** Cada versión crea un archivo nuevo en `/app/storage/{empresa}/{documentoId}/{versionTag}/`.
4. **Solo una versión Vigente por documento.** Al aprobar una nueva, la anterior pasa a OBSOLETO con `FueAprobada = true`.
5. **SemVer exacto:** creación = v1.0.0; rechazo menor = patch+1; rechazo mayor = minor+1; aprobación = major+1. (En `SemVerService`.)
6. **Todo movimiento se audita en PostgreSQL.** Ninguna acción de escritura sin registro.
7. **RBAC:** SuperAdmin(0) > Admin(1) > Revisor(2) > Creador(3) > Lector(4). El Lector solo ve/descarga versiones vigentes.
8. **Secretos solo en `.env`.** Nunca en `appsettings.json` versionado ni hardcodeados.
9. **Nunca `DELETE` físico** de documentos/usuarios: usa baja lógica (`EliminadoEn`).

## 3. Estados de versión
`BORRADOR → EN_REVISION → APROBADO(Vigente) | RECHAZADO`; al aprobar otra, la previa → `OBSOLETO` (FueAprobada=true).

## 4. Flujo de trabajo del agente
1. Lee `AGENTS.md` + la sección relevante de `docs/DISENO.md`.
2. Localiza el archivo real antes de cambiar; no dupliques servicios existentes.
3. Cambios pequeños y verificables. Tras tocar C#: `dotnet build` y `dotnet test`. Tras tocar compose: `docker compose config`.
4. Si cambias el modelo de datos, actualiza **a la vez** la entidad EF, la migración y el script SQL/Mongo correspondiente.
5. No marques una tarea como terminada si el build o los tests fallan.
6. Puerto o secreto nuevo → refléjalo en `.env.example` y `docker-compose.yml`.

## 5. Convenciones
- **.NET 10**, C# moderno. Proyecto en `QualityDoc/QualityDoc/` (namespace `QualityDoc`). Identificadores en inglés; comentarios/commits en español.
- PascalCase clases/métodos; camelCase locales; interfaces con `I`.
- EF Core con migraciones. Auth por cookies. Contraseñas con BCrypt.
- Git: Conventional Commits (`feat:`, `fix:`, `docs:`, `db:`, `docker:`). Ramas `main`, `develop`, `feature/*`.

## 6. Errores a evitar
- ❌ Meter lógica en controllers/vistas. ✅ En `Services/`.
- ❌ Query sin filtro de empresa → fuga entre tenants. ✅ Global Query Filter.
- ❌ Sobrescribir archivos o versiones. ✅ Path único por versión.
- ❌ Secretos en `appsettings`. ✅ `.env`.
- ❌ Añadir Redis/RabbitMQ u otro motor de datos (regla de 3 motores). ✅ Integración por API HTTP o BD.
- ❌ Que .NET escriba en Mongo o extraiga metadatos. ✅ Eso lo hace SOLO Node; .NET notifica por HTTP.
- ❌ Meter lógica de creación/edición en el portal PHP. ✅ PHP es solo lectura + reportes (Postgres).
