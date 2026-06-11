# ROADMAP — Brands AI Engine + Fashion Signal en Open Carrusel

> **Última actualización:** 2026-06-10
> **Estado:** Discovery + Planning (sin código todavía)
> **Backup persistente:** engram (`mem_search "ROADMAP Brands AI Engine"`)

---

## Hallazgo crítico

**Open Carrusel YA ES un motor de generación de carruseles completo.** El `chat-system-prompt.ts` ya define:

- Estructura de 8 slides (HOOK → Setup → Value → CTA)
- Inyección automática de Brand (logo + colors + fonts + keywords)
- Soporte de Style Presets reutilizables (`designRules` + `exampleSlideHtml`)
- Comandos curl para crear/editar/borrar slides

**NO HAY QUE CONSTRUIR EL MOTOR.** Hay que:

1. Configurarlo con la identidad de Brands AI Engine
2. Conectar el Fashion Signal como input automático
3. Enriquecer el system prompt con Brand DNA (buyer personas, tono)

---

## Arquitectura — Brand DNA como capa transversal

El Brand DNA **NO es una caja secuencial** — alimenta 3 cajas en paralelo:

```
┌─ BRAND DNA "Brands AI Engine" ─────────────────────┐
│  - positioning, mission, values                    │
│  - buyerPersonas [{ name, age, pains, goals, lang }]│
│  - toneOfVoice { personality, doSay, dontSay }     │
│  - visualIdentity { colors, fonts, logo, imagery } │
└────────┬─────────────┬─────────────┬───────────────┘
         │             │             │
         ▼             ▼             ▼
   FASHION       BRAND CONFIG   AGENTE CLAUDE
   SIGNAL        (visual)       (copy + tono)
   (research)
         ↓
   8 slides estilo Brands AI Engine
```

### Decisión arquitectónica — Opción C + B

- **C — IMPORTAR** el DNA desde el sistema existente (brand-analyzer o ads-dna), no reinventarlo
- **B — SEPARAR** entidades: `BrandConfig` (visual, ya existe) vs `BrandDNA` (estratégico, nuevo en `data/brand-dna.json`)

---

## Pipeline end-to-end

```
                    ┌──────────────────┐
                    │   BRAND DNA      │ ← se carga 1 vez
                    │   "Brands AI     │   (o cuando cambia
                    │    Engine"       │    la estrategia)
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
       ┌──────────┐   ┌──────────┐   ┌──────────────┐
       │ FASHION  │   │  BRAND   │   │  AGENTE      │
       │ SIGNAL   │──▶│  CONFIG  │──▶│  CLAUDE      │
       │ (research)│   │ (visual) │   │  (copy+slide)│
       └──────────┘   └──────────┘   └──────┬───────┘
                                            │
                                            ▼
                                   ┌────────────────┐
                                   │  8 slides      │
                                   │  estilo BAE    │
                                   └────────────────┘
```

### Flujo semanal detallado

1. **LUNES 9:00 AM** — cron dispara Fashion Signal
2. Signal hace research web usando el "libro de contenido" + buyer personas del DNA
3. Output JSON con `{ week, topics: [{ hook, angle, keywords, image? }] }`
4. `POST /api/signal/sync` en Open Carrusel recibe el JSON
5. Por cada topic → crea Carousel + inyecta topic al agente Claude
6. Agente Claude genera 8 slides usando Brand + Style Preset + DNA
7. Storage atomic en `data/carousels.json`
8. Usuario revisa en UI (`localhost:3000`)
9. Export PNG ZIP a 1080x1350
10. Publicación manual en Instagram

---

## Estado actual

### Existe (no tocar)

- Motor de slides (`src/lib/chat-system-prompt.ts`)
- `BrandConfig` type + UI de setup (`src/types/brand.ts` + `src/components/brand/BrandSetup.tsx`)
- Soporte de `logoPath` en `BrandConfig`
- Style Presets con `designRules` + `exampleSlideHtml`
- Storage atomic con `async-mutex` (`src/lib/data.ts`)
- Export PNG ZIP con Puppeteer
- Chat history en localStorage
- Reference images (Read tool del agente)

### Falta construir

- Brand DNA type + storage (`data/brand-dna.json`)
- Inyección del DNA en el system prompt
- Endpoint `POST /api/signal/sync`
- Cliente del Fashion Signal (`src/lib/signal/client.ts`)
- Transformador Signal → Carousel (`src/lib/signal/transformer.ts`)
- Style Preset "Brands AI Engine Fashion" (crear UNO via UI o seed)
- UI: botón "Sync from Fashion Signal" en TopBar

### Hay que configurar (no construir)

- Subir logo Brands AI Engine vía UI
- Configurar Brand colors (`#F5D567` como accent/primary)
- Configurar fonts (sans bold + serif italic)

---

## Roadmap por fases

### FASE 0 — Discovery (30 min) — read-only

- [ ] Explorar `/Users/fedetanco/Developments/approach-projects/approach-content/brands-ai-engine-carousels/`
      Comandos: `fd -d 3`, `rg "BRAIEN"`, `rg "gpt-image|fal"`
- [ ] Localizar el repo de generación de DNA (preguntar al user)
- [ ] Ver shape del `brand-profile.json` existente
- [ ] Localizar fork de Fashion Signal
- [ ] Si el Signal ya corrió, ver el output real para diseñar el ingest

### FASE 1 — Visual Foundation (45 min)

- [ ] Recibir logo Brands AI Engine del usuario
- [ ] Configurar Brand en UI: name, colors, fonts, logo, styleKeywords
- [ ] Crear Style Preset "Brands AI Engine Fashion":
  - `designRules`: descripción detallada del estilo visual
  - `exampleSlideHtml`: una portada bien hecha (HTML completo)
- [ ] Testear: crear carrusel de prueba y validar que el agente respeta el estilo

### FASE 2 — Brand DNA Infrastructure (1-2 hs)

- [ ] Crear `src/types/brand-dna.ts` con shape rico (personas, voz, valores)
- [ ] Crear `src/lib/brand-dna.ts` con read/write (analog a `data.ts` patterns)
- [ ] Crear endpoint `GET/PUT /api/brand-dna`
- [ ] Extender `chat-system-prompt.ts` para inyectar DNA en el prompt:
  - Buyer personas → guía de a quién hablarle
  - Tone of voice → guía de cómo hablarle
  - Do/Don't say → vocabulario permitido y prohibido
- [ ] Decisión: ¿import desde `brand-profile.json` externo o edición directa en UI?
      **Recomendación:** ambos — endpoint `POST /api/brand-dna/import` que reciba el JSON externo

### FASE 3 — Signal Ingest (2-3 hs)

- [ ] Crear `src/lib/signal/client.ts` — wrapper del fetch al Signal
      Config vía env: `FASHION_SIGNAL_URL`, `FASHION_SIGNAL_TOKEN`
- [ ] Crear `src/lib/signal/transformer.ts` — mapear shape del Signal a topics para el agente
- [ ] Crear `src/app/api/signal/sync/route.ts`:
  - Body opcional: `{ since?: ISO date }`
  - Por cada topic del Signal: crea Carousel + inyecta topic al chat del agente
- [ ] UI: componente `SignalSyncButton` en TopBar
- [ ] Decisión pendiente: ¿content-engine entre Signal y Open Carrusel? Decidir VIENDO el output real del Signal

### FASE 4 — Automatización (1 hs)

- [ ] Cron semanal (Vercel cron o GitHub Actions) que dispare `POST /api/signal/sync`
- [ ] Notificación al usuario cuando se generan carruseles nuevos (opcional)

### FASE 5 — Refinamiento (post-MVP)

- [ ] Métricas: cuántos carruseles del Signal terminan publicados sin edición
- [ ] Mejora del DNA basado en lo que el usuario edita más
- [ ] Versionado del DNA (audit log)

---

## Decisiones pendientes del usuario

| ID | Pregunta | Estado |
|----|----------|--------|
| **D1** | Logo: ¿formato (SVG/PNG)? ¿color? ¿posición en portada? | Pendiente |
| **D2** | Ubicación del sistema de DNA actual (¿brands-ai-engine? ¿approach-content? ¿skill brand-analyzer?) | Pendiente |
| **D3** | ¿DNA de Brands AI Engine ya existe en algún `brand-profile.json` o lo generamos hoy? | Pendiente |
| **D4** | DNA local (copia en open-carrusel) o referencia por URL al sistema central | Pendiente |
| **D5** | Fashion Signal: ¿dónde vive el fork? ¿ya corrió? ¿shape del output? | Pendiente |
| **D6** | "Libro de contenido" del Signal: ¿MD/Notion/JSON? ¿dónde vive? | Pendiente |
| **D7** | ¿content-engine sí o no? Decidir DESPUÉS de ver output real del Signal | Bloqueado por D5 |
| **D8** | Texto en portadas: HTML overlay (recomendado) vs Fal | Recomendado HTML, falta confirmar |

---

## Riesgos y gotchas

| ID | Riesgo | Mitigación |
|----|--------|------------|
| **R1** | Inflar el system prompt con TODO el DNA | Incluir solo persona target + tono + brand visual; el resto bajo demanda |
| **R2** | Sincronización del DNA (drift entre central y local) | Endpoint `/import` + timestamp de última sync |
| **R3** | Style Preset puede competir con DNA (ambigüedad visual) | Preset SOLO describe layout/composición; DNA tiene assets (colors/fonts/logo) |
| **R4** | Paleta amarilla varía entre las 4 muestras | Usar eyedropper antes de fijar `#F5D567` |
| **R5** | Sandbox del iframe bloquea JS | Todos los slides en CSS puro (ya documentado en CLAUDE.md) |
| **R6** | Hotlink vs descarga de imágenes externas del Signal | Recomendado descarga local a `/uploads/` |
| **R7** | content-engine como overhead innecesario | NO meterlo hasta validar shape del Signal |

---

## Qué NO hacer

- ❌ NO escribir código antes de explorar `brands-ai-engine-carousels/` (alto riesgo de duplicar)
- ❌ NO meter content-engine sin validar shape del Signal primero
- ❌ NO crear el template Brands AI Engine con valores asumidos — pedir confirmación de colores y fuentes
- ❌ NO mezclar Brand DNA en `BrandConfig` — separar entidades (Opción B)
- ❌ NO regenerar el DNA si ya existe en otro proyecto (Opción C — importar)
- ❌ NO empezar la UI antes de tener el endpoint funcionando con `curl`
- ❌ NO usar Fal para texto si HTML overlay alcanza
- ❌ NO inflar el system prompt con todo el DNA — selección selectiva

---

## Próximo paso inmediato

Esperando respuestas del usuario a **D1, D2, D3, D5, D6** antes de tocar código.

Mientras tanto, lo único productivo a hacer es **FASE 0** (discovery read-only) — explorar `brands-ai-engine-carousels/`.

---

## Aclaraciones de naming

- **Brands AI Engine** = nombre del producto/proyecto (correcto)
- **BRAIEN** = wordmark visible en las portadas viejas (deprecado — ahora reemplazado por logo)

---

## Referencias

### Repos relacionados

- `/Users/fedetanco/Developments/approach-projects/open-carrusel` (este repo)
- `/Users/fedetanco/Developments/approach-projects/approach-content/brands-ai-engine-carousels/` (referencia visual, por explorar)
- Fork de signal-dashboard para fashion (path pendiente)
- Sistema de generación de DNA (probablemente `brands-ai-engine` o `approach-content`)

### Memorias relacionadas en engram

- `pattern/estilo-visual-braien-para-portadas-de-carrusel` — patrón visual de portadas
- `architecture/pipeline-completo-fashion-signal-open-carrusel` — pipeline alto nivel
- `decision/nombre-correcto-brands-ai-engine-no-braien` — naming
- `decision/cambio-logo-en-vez-de-wordmark-braien-en-portadas` — logo reemplaza wordmark
- `decision/roadmap-brands-ai-engine-fashion-signal-en-open-carrusel` — este roadmap

### Archivos clave del repo

- `src/lib/chat-system-prompt.ts` — Motor de generación (ya existe)
- `src/types/brand.ts` — `BrandConfig` actual (visual nomás)
- `src/lib/data.ts` — Storage atomic con async-mutex
- `src/lib/slide-html.ts` — `wrapSlideHtml()` para iframe sandbox
- `CLAUDE.md` — convenciones del proyecto
