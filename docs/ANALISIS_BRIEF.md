# Análisis del brief — ValorInmueble Santander

Documento analizado: *Brief de desarrollo — ValorInmueble Santander* (21 secciones).
Estado del repositorio al momento del análisis: vacío (sin código ni commits previos).

---

## 1. Resumen ejecutivo

El brief describe un **MVP de captación de propietarios** disfrazado de herramienta gratuita: el usuario valora su
inmueble, recibe una cifra orientativa y se le ofrece una valoración profesional. El producto real es el **lead**, y el
éxito se mide por cuántos propietarios de Bucaramanga, Floridablanca, Girón y Piedecuesta terminan hablando por
WhatsApp con Inmobiliaria Ramírez Hernández.

El documento está bien enfocado: define embudo, prioridades, criterio de éxito y pide explícitamente **no usar IA** en
el motor inicial. Sin embargo, tiene **inconsistencias entre secciones** (formulario vs. base de datos vs. motor) y deja
abiertas decisiones de negocio que deben confirmarse antes de programar.

## 2. Fortalezas

| Aspecto | Por qué es un acierto |
|---|---|
| Embudo explícito (§4, §11) | Da una métrica clara por etapa y guía el diseño del CRM. |
| Motor por reglas primero (§12) | Es explicable, auditable y ajustable sin datos históricos. Evita prometer precisión que no existe. |
| Parámetros editables sin código (§12) | Permite que el equipo comercial calibre precios por m² con su experiencia real. |
| Prioridad del MVP (§17) | Orden correcto: primero lo que genera leads, luego lo operativo. |
| Aviso legal (§7) | Imprescindible: en Colombia el avalúo comercial lo firma un avaluador inscrito en el RAA (Ley 1673 de 2013). |
| Marca respaldada por la inmobiliaria (§19) | Aprovecha la reputación existente sin arriesgar la marca principal. |

## 3. Inconsistencias y vacíos detectados

### 3.1 El esquema de base de datos (§13) no cubre el formulario (§6)

| Campo del formulario | ¿Está en `properties`? |
|---|---|
| Área construida / Área privada | Solo existe `area` (una) |
| Estrato | **No** |
| Ascensor, balcón, terraza, conjunto cerrado, amoblado | **No** |
| Antigüedad | **No** |
| Piso | Sí (`floor`) |
| Estado | Sí (`condition`) |

Además:
- `valuations` no guarda el **valor puntual** (solo min/max), ni los **factores aplicados**, ni la **versión de
  parámetros** usada. Sin eso no se puede explicar un resultado antiguo ni auditar cambios de precios por m².
- `leads` no tiene `address`, `preferred_contact_time` (pedidos en §8), ni **consentimiento de tratamiento de datos**,
  ni `valuation_id`, ni UTM/origen de campaña.
- No hay tabla para **precios por m² por barrio** ni para **factores de ajuste**, que son justamente lo que el panel
  (§12) debe editar.
- No hay tabla de **eventos de analytics** (§15), **historial de estados del lead**, **visitas** ni **contratos**, que
  aparecen en el dashboard interno (§11).
- `users` no contempla credenciales/autenticación (§16).

### 3.2 El motor de valoración (§12) no define cómo calcular el arriendo

Se describe la venta (`precio m² × área × ajustes`) pero no el **arriendo**. Hay dos opciones:
1. Precio de arriendo por m² por barrio (más preciso, requiere más datos).
2. Arriendo = valor de venta × tasa de rentabilidad mensual del sector (más simple, un solo parámetro por zona).

Recomendación: opción 2 para el MVP, con posibilidad de sobreescribir por barrio.

### 3.3 El estrato no aparece entre los factores de ajuste

En el área metropolitana de Bucaramanga el estrato es de los predictores más fuertes del precio. Se pide en el
formulario pero no se usa en §12. Debe ser, como mínimo, un factor o una dimensión del precio base por m².

### 3.4 Área construida vs. área privada

El brief pide ambas pero no dice cuál es la base del cálculo. Hay que definirlo y ser coherente con los datos de
precio por m² que aporte la inmobiliaria.

### 3.5 Objetivos "comprar" e "invertir" en el formulario de valoración

El formulario asume que el usuario tiene un inmueble. Para quien quiere comprar/invertir el flujo tiene sentido solo si
valora un inmueble que está evaluando. Conviene tratarlos como leads de otro tipo (comprador/inversionista) y no
mezclarlos con captación de propietarios.

### 3.6 Protección de datos personales (§16) queda genérica

Para cumplir la **Ley 1581 de 2012** y el **Decreto 1377 de 2013** el MVP necesita, desde el día uno:
- Casilla de **autorización expresa** de tratamiento de datos (no premarcada) antes de enviar el lead.
- **Política de tratamiento de datos** publicada, con responsable, finalidades y canal para ejercer derechos
  (conocer, actualizar, rectificar, suprimir).
- Guardar la **prueba de la autorización** (fecha, versión de la política, IP).
- Banner de **consentimiento de cookies** antes de cargar Google Analytics / Meta Pixel.

### 3.7 Datos de mercado

El valor de la herramienta depende de los precios por m². El brief no indica de dónde saldrán. Los datos de
demostración deben marcarse claramente como ficticios y **no publicarse en producción** hasta calibrarlos con datos
reales de la inmobiliaria.

## 4. Riesgos

| Riesgo | Impacto | Mitigación |
|---|---|---|
| Estimaciones muy alejadas del mercado | Pérdida de credibilidad, leads molestos | Rangos amplios, factores explicados, calibración con cierres reales, aviso legal visible |
| Que parezca un avalúo oficial | Riesgo legal | Lenguaje "estimación orientativa", nunca "avalúo" |
| Formulario largo (19 campos) | Abandono en móvil | Pasos cortos, campos opcionales avanzados, barra de progreso, medir abandono por paso |
| Pedir datos de contacto antes del resultado | Baja conversión / desconfianza | Mostrar resultado primero y pedir datos para la valoración profesional (tal como plantea el brief) |
| Spam en el formulario de leads | CRM contaminado | Rate limiting, honeypot, validación de teléfono colombiano |
| Alcance creciente (dashboard propietario, pagos, contratos) | Retraso del MVP | Solo crear tablas preparatorias; no construir esas pantallas en fase 1 |

## 5. Propuesta técnica (para validar)

**Stack sugerido**
- **Next.js (App Router) + TypeScript**: SSR/SSG para SEO local, API routes, un solo proyecto.
- **Tailwind CSS**: UI mobile-first rápida.
- **PostgreSQL + Prisma**: esquema tipado y migraciones (SQLite en desarrollo local si se prefiere).
- **Auth.js (NextAuth)** con credenciales y roles `admin`, `agent`, `owner`.
- **Zod**: validación compartida cliente/servidor.
- Despliegue: Vercel + Postgres gestionado (Neon/Supabase), o un VPS si la inmobiliaria ya tiene uno.

**Rutas**

| Ruta | Descripción |
|---|---|
| `/` | Landing con hero, CTA y accesos secundarios |
| `/valorar` | Formulario por pasos |
| `/resultado/[id]` | Tarjeta de resultado + CTA valoración profesional |
| `/gracias` | Confirmación + botón WhatsApp |
| `/rentabilidad` | Calculadora de inversionista (fase 2) |
| `/valor-inmueble/[ciudad]` y `/valor-inmueble/[ciudad]/[barrio]` | Páginas SEO locales |
| `/politica-de-datos` | Política de tratamiento de datos |
| `/admin` | Resumen y embudo |
| `/admin/leads`, `/admin/leads/[id]` | CRM de leads |
| `/admin/valoraciones`, `/admin/inmuebles`, `/admin/propietarios` | Listados |
| `/admin/configuracion/precios`, `/admin/configuracion/factores` | Edición de precios/m² y factores |

**Algoritmo inicial de valoración (reglas)**

```
base_m2     = precio_m2(ciudad, barrio, tipo)          // editable en admin; si no hay barrio, promedio de la ciudad
valor_base  = base_m2 × área_referencia
ajuste      = Π factores(estrato, habitaciones, baños, parqueaderos, piso+ascensor,
                         balcón, terraza, conjunto, estado, antigüedad, amoblado)
venta       = valor_base × ajuste
rango_venta = venta × (1 ± margen)                     // margen mayor si no hay precio del barrio o pocos comparables
arriendo    = venta × tasa_arriendo_mensual(zona)      // editable; sobreescribible por barrio
rango_arr   = arriendo × (1 ± margen)
rent_bruta  = arriendo × 12 / venta × 100
```

- Cada factor es un multiplicador configurable (p. ej. 1.00 neutro, 1.05 +5 %).
- Se limita el ajuste total a un rango (p. ej. 0.70–1.30) para evitar resultados absurdos.
- El resultado guarda los factores aplicados para mostrar "factores que influyeron" y para auditoría.
- Cuando existan comparables suficientes del mismo barrio/tipo, el precio base puede pasar a ser la mediana del
  precio/m² de esos comparables (arquitectura preparada desde el inicio).

**Esquema de datos (ajustado)** — añade a lo propuesto en §13:
- `properties`: `built_area`, `private_area`, `stratum`, `has_elevator`, `has_balcony`, `has_terrace`,
  `is_gated_community`, `is_furnished`, `age_years`.
- `valuations`: `estimated_sale`, `estimated_rent`, `factors_json`, `pricing_version`, `input_json`.
- `leads`: `address`, `preferred_contact_time`, `valuation_id`, `data_consent_at`, `consent_policy_version`,
  `utm_source/medium/campaign`, `assigned_to`.
- Nuevas: `neighborhoods` (ciudad, barrio, estrato típico, precio m² venta, tasa de arriendo), `valuation_factors`,
  `lead_status_history`, `analytics_events`.
- `documents`, `payments`, `maintenance_requests`: solo se crean, sin pantallas en el MVP.

**Leads**
1. Se guarda en BD con estado `NUEVO` y la valoración asociada.
2. Se abre WhatsApp (`https://wa.me/57XXXXXXXXXX?text=...`) con mensaje precargado que incluye barrio, tipo, valor
   estimado y el ID del lead.
3. (Opcional) Notificación por correo al equipo.
4. En el admin, el lead avanza por el embudo con historial de cambios.

## 6. Supuestos que deben confirmarse

1. **Nombre de marca y dominio** definitivos (ValorInmueble Santander / ValoraSantander / MiValorInmueble).
2. **Número de WhatsApp** comercial y texto del mensaje precargado.
3. **Fuente de precios por m²** por barrio y tipo de inmueble, y **tasa de arriendo** por zona. ¿La inmobiliaria
   puede entregar una hoja inicial con sus cierres recientes?
4. **Listado de barrios** por municipio que se habilitarán al lanzamiento.
5. **Área base** del cálculo: construida o privada.
6. **Tipos de inmueble** del MVP (¿solo apartamento y casa? ¿locales, lotes, oficinas?).
7. **Hosting** y presupuesto (Vercel/Neon gratuitos vs. servidor propio).
8. **Responsable del tratamiento de datos**: razón social, NIT, correo para habeas data.
9. ¿Se muestra el resultado **antes** de pedir datos de contacto? (el brief sugiere que sí; es lo recomendado).
10. ¿Quién usa el admin y con qué roles? ¿Habrá asesores con leads asignados?
11. IDs de **Google Analytics 4** y **Meta Pixel** (si ya existen).
12. Identidad visual: logo, colores y tipografía de la inmobiliaria.

## 7. Plan de implementación incremental sugerido

1. **Base del proyecto**: Next.js + Tailwind + Prisma + esquema + datos de demostración.
2. **Motor de valoración** como módulo puro con pruebas unitarias.
3. **Landing + formulario por pasos + resultado** (mobile-first).
4. **Captura de lead + consentimiento + WhatsApp**.
5. **Admin**: login, leads, embudo, edición de precios/m² y factores.
6. **SEO** (metadata, OG, sitemap, robots, páginas locales) y **analytics** con consentimiento.
7. Documentación para editar barrios, precios, factores, contacto y SEO.

Criterio de éxito (del brief): un propietario entra desde el celular, valora en pocos pasos, entiende el resultado y
escribe por WhatsApp; la inmobiliaria ve y gestiona ese lead en el dashboard.
