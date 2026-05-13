# QA Test Plan — Bolivia Booking Go
**Materia:** Quality Assurance
**API:** Restful-Booker | **Herramienta:** Postman + Newman
 
---
 
## Contexto
 
La startup **Bolivia Booking Go** necesita certificar su API de reservas antes del lanzamiento. El proceso debe ser **completamente desatendido (CI/CD Ready)**: sin intervención manual, con seguridad y datos manejados dinámicamente.
 
- **Documentación API:** https://restful-booker.herokuapp.com/apidoc/index.html  
- **Base URL:** `https://restful-booker.herokuapp.com`  
---
 
## Alcance
 
**Dentro del alcance:**
- Autenticación dinámica con token en variable de entorno (HU-01)
- Ciclo de vida completo de reserva: Crear, Consultar, Actualizar, Eliminar (HU-02)
- Validación masiva con 5 casos desde archivo JSON externo (HU-03)
- Consulta por ID con validación de esquema y tiempo de respuesta < 800ms (HU-04)
- Actualización parcial de campos con token de seguridad (HU-05)
- Cancelación con auditoría post-eliminación confirmando 404 (HU-06)
- Listado filtrado por nombre con validación de array no vacío (HU-07)
**Fuera del alcance:** pruebas de carga, seguridad ofensiva, pruebas de UI.
 
---
 
## Variables de Entorno
 
| Variable | Se genera en | Se usa en | Se limpia en |
|---|---|---|---|
| `base_url` | Manual | Todos los requests | — |
| `auth_token` | HU-01 Auth | HU-02 Update/Delete · HU-05 · HU-06 | — |
| `booking_id` | HU-02 CreateBooking | HU-02 Get/Update/Delete · HU-04 · HU-05 | HU-02 Delete · HU-06 Delete |
| `firstname` | Booking_Data.json (HU-03) | HU-07 Query Param | — |
 
---
 
## Orden de Ejecución
 
```
1.  POST   /auth               → HU-01 | genera auth_token
2.  POST   /booking            → HU-02 | genera booking_id
3.  GET    /booking/:id        → HU-02 · HU-04 | usa booking_id
4.  PUT    /booking/:id        → HU-02 | actualiza nombre (auth_token)
5.  PUT    /booking/:id        → HU-05 | actualización parcial (auth_token)
6.  DELETE /booking/:id        → HU-02 | elimina reserva (auth_token)
7.  POST   /booking            → HU-06 | nueva reserva para auditoría
8.  DELETE /booking/:id        → HU-06 | elimina + verifica 404 (auth_token)
9.  POST   /booking × 5        → HU-03 | Collection Runner con Booking_Data.json
10. GET    /booking?firstname= → HU-07 | filtra por nombre
```
 
---
 
## Casos de Prueba
 
| ID | HU | Descripción | Precondición | Resultado Esperado |
|---|---|---|---|---|
| TC-01 | HU-01 | Generar token con credenciales válidas | API disponible | 200 OK · `auth_token` guardado |
| TC-02 | HU-02 | Crear reserva y persistir bookingid | TC-01 exitoso | 200 OK · `booking_id` guardado |
| TC-03 | HU-02 | Consultar reserva por ID | TC-02 exitoso | 200 OK · campos presentes |
| TC-04 | HU-02 | Actualizar nombre del huésped | TC-01 + TC-02 | 200 OK · firstname = "James" |
| TC-05 | HU-02 | Eliminar reserva | TC-01 + TC-02 | 201 Created · `booking_id` removido |
| TC-06 | HU-03 | Crear 5 reservas desde JSON externo | Runner + archivo cargado | 5 × 200 OK · 5 × bookingid |
| TC-07 | HU-04 | Consulta con esquema y tiempo < 800ms | TC-02 exitoso | 200 OK · tiempo < 800ms |
| TC-08 | HU-05 | Actualización parcial con token | TC-01 + TC-02 | 200 OK · campos correctos |
| TC-09 | HU-06 | Eliminar y auditar con GET posterior | TC-01 + reserva nueva | 201 · GET mismo ID → 404 |
| TC-10 | HU-07 | Listar reservas filtrando por firstname | TC-06 exitoso | 200 OK · array no vacío |
 
---
 
## Tabla de Decisiones
 
| HU | Condición | Método | Status Esperado | Resultado |
|---|---|---|---|---|
| HU-01 | Credenciales válidas `admin/password123` | POST | 200 OK | ✅ PASS — token guardado |
| HU-01 | Credenciales inválidas | POST | 200 OK | ❌ FAIL — abortar colección |
| HU-01 | Token expirado (~10 min) | — | — | ⚠️ Re-ejecutar HU-01 |
| HU-02 | Body completo válido | POST | 200 OK | ✅ PASS — booking_id guardado |
| HU-02 | Body incompleto o malformado | POST | 400 / 500 | ❌ FAIL — booking_id no persiste |
| HU-02 | `booking_id` válido en entorno | GET | 200 OK | ✅ PASS — campos validados |
| HU-02 | `booking_id` vacío o undefined | GET | 404 Not Found | ❌ FAIL — ejecutar CreateBooking |
| HU-02 | Token válido + booking_id válido | PUT | 200 OK | ✅ PASS — nombre actualizado |
| HU-02 | Token expirado | PUT | 403 Forbidden | ❌ FAIL → Re-autenticar |
| HU-02 | Token válido + booking propio | DELETE | 201 Created | ✅ PASS — booking_id limpiado |
| HU-02 | Token expirado o booking ajeno | DELETE | 403 Forbidden | ❌ FAIL → Re-autenticar |
| HU-03 | 5 objetos válidos en Runner | POST | 200 OK × 5 | ✅ PASS × 5 |
| HU-03 | Ejecutado con Send (sin Runner) | POST | 200 OK | ❌ FAIL — iterationData = undefined |
| HU-04 | booking_id válido · servidor rápido | GET | 200 OK · < 800ms | ✅ PASS |
| HU-04 | Servidor lento > 800ms | GET | 200 OK · > 800ms | ⚠️ PERF FAIL |
| HU-05 | Token válido + body con cambios | PUT | 200 OK | ✅ PASS — campos verificados |
| HU-05 | Body parcial sin campos obligatorios | PUT | 400 Bad Request | ❌ FAIL — enviar body completo |
| HU-05 | Token expirado | PUT | 403 Forbidden | ❌ FAIL → Re-autenticar |
| HU-06 | Token válido + booking_id recién creado | DELETE | 201 Created | ✅ PASS |
| HU-06 | GET al ID eliminado | GET | 404 Not Found | ✅ AUDIT PASS |
| HU-07 | `?firstname=Alice` (registrado en HU-03) | GET | 200 OK | ✅ PASS — array no vacío |
| HU-07 | Nombre no existente en la API | GET | 200 OK | ❌ FAIL — array vacío |
 
---
 
## Análisis de Riesgos
 
| Riesgo | Nivel | Impacto | Mitigación |
|---|---|---|---|
| Token expira en ~10 minutos | 🔴 ALTO | 403 en Update, Delete, PATCH | Re-ejecutar HU-01 antes de cada sesión |
| API pública — IDs de otros usuarios | 🔴 ALTO | 403 al modificar reservas ajenas | Siempre usar `booking_id` de la sesión actual |
| Servidor lento o caído (Heroku) | 🟡 MEDIO | Timeouts · fallo en test de 800ms | Reintentar en horarios de menor tráfico |
| DELETE borra `booking_id` antes de HU-06 | 🟡 MEDIO | HU-06 no tiene ID para eliminar | HU-06 crea su propia reserva antes del DELETE |
| PUT parcial devuelve 400 con body incompleto | 🟡 MEDIO | HU-05 no cumple criterio PATCH puro | Enviar body completo modificando solo campos requeridos |
| HU-03 ejecutada con Send en vez de Runner | 🟢 BAJO | `pm.iterationData` retorna undefined | Usar siempre Collection Runner con archivo JSON |
 
---
 
## Comando Newman
 
```bash
# Instalar Newman y reporter htmlextra
npm install -g newman newman-reporter-htmlextra
 
# Ejecutar la colección completa con reporte HTML
newman run Booking_Collection.json \
    -e Booking_Environment.json \
    -d Booking_Data.json \
    --reporters htmlextra \
    --reporter-htmlextra-export Reporte_Final.html \
    --reporter-htmlextra-title "Bolivia Booking Go — QA Report"
```
 
---
