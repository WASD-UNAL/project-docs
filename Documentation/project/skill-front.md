# Revisión de código — TypeScript + React 19 (`project-frontend`)

Revisión completa de `src/` (10 archivos .ts/.tsx) usando los criterios de la skill `typescript-react-reviewer`.

## Resumen

El proyecto es un landing page estático (Vite + React 19 + TypeScript + Tailwind v4). El código es pequeño, bien organizado y **no presenta hallazgos críticos ni de alta prioridad**. No hay `useEffect` para estado derivado, no hay mutaciones directas de estado, no hay `key={index}`, no hay `any`, no hay mal uso de hooks de React 19 (`useFormStatus`, `use()`, etc. — de hecho no se usan).

| Severidad | Hallazgos |
|---|---|
| 🚫 Crítico | 0 |
| ⚠️ Alta prioridad | 0 |
| 📝 Arquitectura/Estilo | 3 (menores, opcionales) |
| ✅ Buenas prácticas observadas | varias (ver abajo) |

---

## 📝 Arquitectura / Estilo (no bloqueantes)

### 1. `tsconfig.app.json` y `tsconfig.node.json` no tienen `"strict": true`

**Archivos:** `tsconfig.app.json`, `tsconfig.node.json`

Ambos configs habilitan flags individuales (`noUnusedLocals`, `noUnusedParameters`, `noFallthroughCasesInSwitch`, `erasableSyntaxOnly`) pero no `strict: true` ni `noUncheckedIndexedAccess`. Hoy esto no causa problemas porque el código no usa `any`, no indexa arrays por posición y no tiene tipos implícitos peligrosos — pero a medida que el proyecto crezca (formularios, fetch a la API del backend, estado), la falta de `strict` permitirá que se introduzcan `any` implícitos y `null`/`undefined` no verificados sin que el compilador avise.

**Recomendación:**
```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "exactOptionalPropertyTypes": true
  }
}
```

### 2. Acoplamiento implícito entre `Hero.tsx` y CSS (`TYPING_START_MS`)

**Archivo:** `src/components/landing/Hero.tsx:14-17`

El inicio del efecto de "escritura" depende de que termine una animación CSS (`animation-delay: 1.2s` + `duración 1.1s` ≈ 2.3s), y eso está sincronizado manualmente con la constante `TYPING_START_MS = 2400`. Está documentado con un comentario, lo cual ayuda, pero es un acoplamiento frágil: si alguien cambia el `animation-delay` o la duración en el CSS/Tailwind sin tocar este archivo, el typewriter arrancará desincronizado del resto de la animación. No es necesario cambiarlo ahora, pero si se vuelve a tocar la animación del Hero, conviene revisar ambos lugares juntos.

### 3. `key={feature}` y `key={title}` usando texto como clave

**Archivos:** `src/components/landing/Pricing.tsx:57`, `src/components/landing/About.tsx:60`, `src/components/landing/Schedule.tsx:29`

Se usa el texto (`feature`, `title`, `day`) como `key` en listas estáticas. Esto es válido (no es el anti-patrón de `key={index}` en listas dinámicas, ya que los datos son estáticos y no se reordenan), pero es frágil ante duplicados accidentales en `data/plans.ts` o en los arrays locales — un duplicado generaría un warning de React de claves repetidas. No requiere cambio, solo tenerlo en cuenta si se agregan nuevas features/planes con texto repetido.

---

## ✅ Buenas prácticas observadas

- **`useEffect` usado correctamente** en `Hero.tsx` (`useTypewriter`) y `ScrollBackground.tsx`: ambos son efectos legítimos (temporizadores y listeners del DOM, no estado derivado), con **cleanup correcto** (`clearTimeout`, `removeEventListener`, `cancelAnimationFrame`).
- **Custom hook bien nombrado**: `useTypewriter` sigue la convención `use*` y tiene arrays de dependencias completos.
- **Manipulación directa del DOM vía `ref`** en `ScrollBackground.tsx` para animación ligada al scroll — patrón correcto para evitar re-renders en cada `scroll`/`resize`, con throttling vía `requestAnimationFrame`.
- **Tipado explícito de props** sin `React.FC`: `RivetPlate` define `interface RivetPlateProps` con `ReactNode` y un valor por defecto para `className`.
- **Sin `any`**, sin mutaciones de estado, sin hooks condicionales, sin componentes definidos dentro de otros componentes.
- **Datos tipados** (`data/plans.ts`) con una interfaz `Plan` clara, reutilizada en `Pricing.tsx`.
- **Componentes pequeños y enfocados** (todos < 105 líneas), buena separación por carpeta (`components/landing`, `pages`, `data`, `utils`).
- `eslint.config.js` ya incluye `eslint-plugin-react-hooks` (flat recommended) y `eslint-plugin-react-refresh`, cubriendo las reglas de hooks más importantes.

---

## Notas de alcance

- No hay manejo de estado global, formularios, ni llamadas a la API del backend (`project-backend`) todavía — por lo tanto no aplica la sección de "State Management" (TanStack Query / Zustand / Jotai) ni los patrones de React 19 para Server Actions (`useActionState`, `useFormStatus`, `use()`). Cuando se integre con el backend (login, planes dinámicos, etc.), vale la pena revisar esos puntos en una nueva pasada.
- No hay Error Boundaries — aceptable para un landing estático sin datos asíncronos, pero recomendable agregarlas cuando se incorpore fetching de datos reales.
