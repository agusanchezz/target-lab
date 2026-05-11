# Sesión de Debugging — Recommendations en Sandbox

**Fecha:** 2026-05-09
**Objetivo:** Implementar Custom Criteria de Recommendations en home de Magento sandbox
**Resultado:** Recomendaciones funcionando end-to-end con Default Template. Design custom queda pendiente.

---

## Contexto del sandbox

- **URL:** `master-7rqtwti-dj2lxi76c6dqk.us-4.magentosite.cloud`
- **Plataforma:** Magento 2 (Adobe Commerce Cloud)
- **at.js:** versión 2.11.7
- **Client code Target:** `improntusptrsd`
- **Property Tags correcta:** `tag para commerce cloud sandbox` (no `m2training.improntus.dev`)

---

## Orden de debugging que funcionó

1. ¿At.js carga? → `typeof adobe.target` === `"object"`
2. ¿Métodos disponibles? → `Object.keys(adobe.target)` y `adobe.target.VERSION`
3. ¿Conexión con servidor Target? → `getOffers()` con `pageLoad` (no `mboxes` para global)
4. ¿La actividad califica? → Mirar `metrics` + `content` en response
5. ¿El criteria tiene resultados? → Estado `Results Ready` vs `Scheduled`
6. ¿El selector del VEC matchea el DOM real? → `document.querySelector('...')`
7. ¿La rule de Tags está pasando los params? → Network tab → request a `tt.omtrdc` → payload

---

## Errores encontrados y soluciones

### 1. `getSettings is not a function`
- **Causa:** Método no disponible en at.js 2.11.7
- **Fix:** Usar `Object.keys(adobe.target)` y `adobe.target.VERSION`

### 2. `global mbox is not allowed in mboxes`
- **Causa:** at.js 2.x no permite pasar `target-global-mbox` en `execute.mboxes`
- **Fix:** Usar `execute.pageLoad` en lugar de `mboxes`

```javascript
adobe.target.getOffers({
  request: {
    execute: {
      pageLoad: {
        parameters: { "entity.id": "home" }
      }
    }
  }
}).then(r => console.log(JSON.stringify(r, null, 2)))
```

### 3. Criteria stuck en `Scheduled` por más de 1 hora
- **Causa:** Formato del CSV no matcheaba el template oficial de Target
- **Fix:** Header debe ser `# Key,Recommendation_1,Recommendation_2,...` (con `#` y espacio)

```csv
# Key,Recommendation_1,Recommendation_2,Recommendation_3,Recommendation_4,Recommendation_5
home,MH09,MH09-L-Blue,MH09-M-Red,MH09-S-Green,MH09-XL-Blue
```

### 4. Recomendaciones no aparecen en home (sin contexto de producto)
- **Causa:** Criteria con `Current Item` como Recommendation Key necesita `entity.id` en el request, pero la home no tiene producto actual
- **Fix:** Pasar `entity.id = "home"` como parámetro fijo desde Tags + usar `home` como keyId en el CSV

### 5. Maximum call stack size exceeded en Tags
- **Causa:** Estaba editando la propiedad equivocada en Tags (`m2training.improntus.dev` en lugar de `tag para commerce cloud sandbox`)
- **Lección:** Verificar siempre la URL del sandbox vs la URL configurada en cada propiedad de Tags

### 6. Orden de actions en Tags
- **Causa:** "Add Params to Page Load Request" estaba después de "Fire page load request"
- **Fix:** El orden correcto es:
  1. Adobe Target v2 - Load Target
  2. Adobe Target v2 - Add Params to Page Load Request
  3. Adobe Target v2 - Fire page load request

### 7. Design custom devuelve `pageLoad: {}` vacío
- **Causas reales identificadas tras leer la doc oficial:**
  1. **`$$entity1.value` es sintaxis Velocity inválida** — usar `\$` o `&#36;` o un prefijo de texto
  2. **`$displayTool.stripTags()` no existe en el runtime de Target** — es de Velocity Tools, librería no incluida
  3. **No usar Silent Reference Notation `$!variable`** — sin esto, las variables undefined se renderizan como literal `$entity.name`
  4. **CRÍTICO: cambios en designs pueden tardar varios minutos en propagarse** — la doc lo dice textualmente. Para probar inmediatamente: crear un design NUEVO en lugar de editar uno existente
- **Sintaxis correcta:**
  - `$!entity.value` (silent ref)
  - `#foreach ($entity in $entities)` (loop preferido cuando hay menos resultados que slots)
  - `${variable}` formal notation cuando hay texto adyacente
- **Referencias clave de la doc:**
  - [Create Design](https://experienceleague.adobe.com/es/docs/target/using/recommendations/recommendations-design/create-design)
  - [Velocity 1.7 User Guide](https://velocity.apache.org/engine/1.7/user-guide.html)
  - [Template FAQ](https://experienceleague.adobe.com/es/docs/target/using/recommendations/recommendations-design/template-faq)

---

## Stack de configuración resultante

### CSV Custom Criteria
- **Ubicación:** `criteria/custom-criteria-sandbox.csv`
- **URL pública:** `https://raw.githubusercontent.com/agusanchezz/target-feed-test/main/criteria/custom-criteria-sandbox.csv`
- **Formato:** Header con `# Key, Recommendation_1...Recommendation_N`

### Recommendation Key
- `Current Item` con keyId fijo `home` pasado desde Tags

### Collection
- `Productos sandbox commerce`

### Tags Rule
- **Property:** `tag para commerce cloud sandbox`
- **Rule:** `Agregar Target V2 - FUNCIONA OK`
- **Event:** Core - Library Loaded (Page Top)
- **Actions (en orden):**
  1. Adobe Target v2 - Load Target
  2. Adobe Target v2 - Add Params to Page Load Request (`entity.id = home`)
  3. Adobe Target v2 - Fire page load request

---

## Pendientes para próxima sesión

- [ ] Investigar por qué los designs custom devuelven `pageLoad: {}` aunque el Default Template funcione
- [ ] Clonar Default Template y modificar CSS de a poco para identificar qué rompe Velocity
- [ ] Mapear el atributo `value` (precio) en el feed/catálogo de Magento — actualmente no existe y muestra `$entity1.value` literal
- [ ] Investigar el error `ProductRecommendationsError: Environment ID is required` — es de Magento Product Recommendations nativo (módulo distinto), revisar si interfiere
- [ ] Configurar prioridades de actividades para evitar conflictos cuando hay múltiples activas en la home
- [ ] Considerar usar `Last Viewed Item` como key para que en sandbox sin tráfico también funcione
- [ ] **Implementar Add to Cart en designs de Recommendations:**
  - Investigar cómo exponer el product ID interno de Magento en el feed (no solo el SKU)
  - Decidir entre form POST a `/checkout/cart/add/` vs AJAX vs evento del datalayer Commerce
  - Resolver el `form_key` (CSRF) en runtime
  - Disparar `cart-data-reload` después del add para refrescar minicart

---

## Comandos útiles guardados para debug

### Verificar at.js cargado
```javascript
typeof adobe.target
Object.keys(adobe.target)
adobe.target.VERSION
```

### Probar getOffers manualmente
```javascript
adobe.target.getOffers({
  request: {
    execute: {
      pageLoad: {
        parameters: { "entity.id": "home" }
      }
    }
  }
}).then(r => console.log(JSON.stringify(r, null, 2)))
```

### Verificar selector del VEC
```javascript
document.querySelector('SELECTOR_DEL_RESPONSE')
```

### Ver versión de librería de Tags cargada
```javascript
document.querySelector('script[src*="launch-"]').src
```
