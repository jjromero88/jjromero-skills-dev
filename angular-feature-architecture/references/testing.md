# Testing

Toda feature se entrega con sus pruebas en verde. Las pruebas se derivan de
la especificación (flujos, validaciones, restricciones y casos de prueba):
cada una provoca un comportamiento desde la pantalla y verifica lo que el
usuario ve (mensaje de campo, notificación, botón deshabilitado) y lo que se
manda a la API.

Verificado en un frontend real (Angular 21 zoneless) el 2026-11-06.

## Runner: Jest + jest-preset-angular

Angular 21 trae Vitest como runner oficial (`@angular/build:unit-test`).
Esta skill usa **Jest** con `jest-preset-angular` (compila componentes y
plantillas con el compilador de Angular sobre jsdom). Al elegir Jest:

- Se desinstala `vitest` y se quita el target `test` de `angular.json`.
- `package.json`: `"test": "jest"`, `"test:watch": "jest --watch"`,
  `"test:coverage": "jest --coverage"`.
- Dependencias de desarrollo (última estable, previo gate de
  `package-security-vetting`): `jest`, `jest-preset-angular`,
  `jest-environment-jsdom`, `@types/jest`. `jsdom` queda (es peer).
- Riesgo a tener presente: `jest-preset-angular` es de terceros y puede ir
  detrás de cada versión mayor de Angular.

`jest.config.mjs` (en la raíz del workspace):

```js
import presets from 'jest-preset-angular/presets/index.js';

/** @type {import('jest').Config} */
export default {
  ...presets.createCjsPreset({ tsconfig: '<rootDir>/tsconfig.spec.json' }),
  setupFilesAfterEnv: ['<rootDir>/setup-jest.ts'],
  testMatch: ['<rootDir>/src/**/*.spec.ts'],
  collectCoverageFrom: ['src/app/**/*.ts', '!src/app/**/*.routes.ts', '!src/app/**/*.model.ts'],
  coverageDirectory: 'coverage',
};
```

`setup-jest.ts` (raíz, fuera de `src/` para que no entre al build de la app):

```ts
// App zoneless: entorno de pruebas sin zone.js
import { setupZonelessTestEnv } from 'jest-preset-angular/setup-env/zoneless';

setupZonelessTestEnv();

// APIs del navegador que jsdom no implementa y que usan componentes compartidos
// (matchMedia, ResizeObserver): stubs mínimos, solo para que rendericen
```

`tsconfig.spec.json`: `"types": ["jest"]`, `"esModuleInterop": true` e
`include` con `setup-jest.ts` y `src/**/*.spec.ts`. No cambies `module` ni
`moduleResolution`: con `node` deja de resolver los `exports` de
`@angular/core/testing`.

## Dónde van

Cada `*.spec.ts` junto al archivo que prueba, con el mismo nombre:
`sede-form.component.ts` → `sede-form.component.spec.ts`;
`geo.helpers.ts` → `geo.helpers.spec.ts`. Sin carpetas `__tests__`.

## Qué se prueba por feature (CRUD)

1. **Formulario** (`{entidad}-form` / `{entidad}-fields`):
   - Cada validación del formulario: dejar el campo vacío, pasar la longitud
     máxima, un formato o rango inválido → se ve el mensaje de error exacto
     bajo el campo y **no** se llama a la API.
   - **Cada validación del backend y de la BD que la especificación nombra**
     (duplicado, en uso, regla de negocio): el servicio de la entidad se
     simula devolviendo el `ApiResponse` de rechazo **con el texto exacto
     del SP** (copiado de la especificación) y se verifica que la pantalla lo
     muestre (notificación o mensaje) y no cierre el formulario. Que la BD
     realmente detecte el caso lo prueba la integración del backend; aquí se
     prueba que la pantalla lo comunique.
   - Camino feliz: el payload que se manda a la API (`toHaveBeenCalledWith`),
     el mensaje de éxito y el evento al padre (`guardado`).
   - Doble envío: mientras guarda, el botón queda deshabilitado.
   - Modo edición: campos inmutables deshabilitados, valores precargados.
   - **Carga al abrir** (obligatoria, `forms-and-validation.md`): en edición
     se llama al `getById` con el ID y el formulario muestra lo que devolvió
     (con datos distintos a los de la fila, para que la prueba falle si se
     usa la fila); al abrir se pide cada select; si el `getById` rechaza, se
     ve el mensaje exacto, el formulario se cierra y el listado se refresca.
2. **Listado / overview:** filtro de búsqueda, estado vacío, acciones visibles
   según permisos, confirmación de eliminación y su resultado.
3. **Servicio de la entidad:** URL y verbo de cada método con
   `provideHttpClient()` + `provideHttpClientTesting()` y
   `HttpTestingController`.
4. **Helpers y pipes:** funciones puras, casos límite.

## Cómo se escriben

- **Componentes con `TestBed`** y la plantilla real. Se simulan con
  `{ provide: Servicio, useValue: { metodo: jest.fn() } }` solo los
  servicios que salen del navegador (API, notificaciones, archivos
  estáticos); los componentes hijos compartidos se renderizan de verdad.
- **Sin zone.js:** después de cada acción, `await fixture.whenStable()`
  (no `fakeAsync`/`tick`, que dependen de zone.js).
- **Inputs con señales:** `fixture.componentRef.setInput('sede', valor)`, o
  un componente anfitrión con señales para probar contenido proyectado.
- **Interactuar como el usuario:** clic en el botón real
  (`button[type="submit"]`), texto en el input y `dispatchEvent(new
  Event('input'))`. Para controles de la librería UI difíciles de operar en
  jsdom (selects, input numérico), el valor se fija en el `FormGroup` que usa
  la plantilla, obtenido desde el DOM, sin tocar miembros protegidos:

  ```ts
  const formulario = () =>
    fixture.debugElement.query(By.directive(FormGroupDirective)).injector.get(FormGroupDirective).form;
  ```

- **Verificar lo visible**: texto de `.campo-error`, llamadas al servicio de
  notificaciones con el mensaje exacto, atributo `disabled` del botón.
- Los mensajes esperados se escriben **literales** (copiados de la
  especificación), nunca importados del código probado.
- **Trazabilidad:** si el proyecto especifica casos de prueba con ID
  (`CP-07`) o validaciones numeradas (`V-03`), el nombre del `it`/`describe`
  empieza con ellos: `it('CP-07 V-03 muestra el mensaje del SP cuando el
  numero ya fue usado', …)`.
- Una prueba en rojo se arregla corrigiendo el **código**, no la prueba,
  salvo que la prueba contradiga la especificación.

## Ejecutar

```bash
npm test                 # todas
npx jest src/app/features/{core}/{entidad}   # una feature
npm run test:coverage
```
