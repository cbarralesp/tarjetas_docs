# 08 · Guía paso a paso: Frontend (Angular 21, arquitectura MVC por features)

Esta guía **registra el proceso completo** con el que se construyó `tarjetas-frontend`. Usa la **estructura de carpetas de `crocante-frontend`** (`core`, `shared`, `features`, `proxy.conf.json`) y la **pantalla de tarjetas Bip!** rojas: resumen, buscador, filtro por estado y formulario aparte para crear y editar.

> Igual que en el backend, cada paso sigue el ritmo **crear → LEVANTAR (`npm start`) → VERIFICAR**.
> Para ver datos reales necesitas el backend levantado (ver la [guía del backend](GUIA-COMPLETA-PASO-A-PASO.md), Paso 6).

## MVC en Angular (organizado por feature)

| MVC | Qué es en Angular | Dónde |
|---|---|---|
| **Model** | Interfaces con la forma de los datos, el *api* que habla con el backend y el *store* con el estado | `core/models`, `core/api`, `features/tarjetas/tarjetas.store.ts` |
| **View** | Las plantillas HTML | `*.html` |
| **Controller** | Las páginas: reciben los eventos del usuario y llaman al store | `tarjetas.page.ts` y `tarjeta-form.page.ts` |

```
src/app
├── core/                         lo que no depende de ninguna pantalla
│   ├── models/                   api-error.ts · tarjeta.model.ts
│   ├── api/                      tarjeta.api.ts            (rutas HTTP del backend)
│   └── interceptors/             error.interceptor.ts      (errores → mensajes legibles)
├── shared/ui/                    piezas reutilizables: clp.pipe · toast(.service) · estado-badge · confirm-dialog
├── features/tarjetas/            la funcionalidad de tarjetas
│   ├── tarjetas.store.ts         estado con signals (Store / Facade)
│   ├── tarjetas.page.*           página del listado ("smart")
│   ├── tarjeta-form.page.*       página del formulario (crear y editar)
│   ├── tarjeta-labels.ts         textos para mostrar
│   └── components/               tarjeta-bip-card  ("tonto")
├── app.ts · app.html · app.css · app.routes.ts · app.config.ts
proxy.conf.json                   reenvía /api hacia el backend (sin CORS)
```

**Regla de oro:** los componentes de `components/` y de `shared/ui` solo **reciben datos** (`input()`) y **emiten eventos** (`output()`). Solo las **páginas** conocen al **store**, y solo el **store** conoce al **api**.

## Mapa del proceso

```
Paso 0  Crear el proyecto y levantarlo
Paso 1  Crear la estructura de carpetas
Paso 2  Models (api-error + tarjeta.model)             → levantar y verificar
Paso 3  Api, interceptor, proxy y configuración HTTP   → levantar y verificar
Paso 4  Shared UI (pipe, toast, badge, diálogo) + estilos → levantar y verificar
Paso 5  Store + textos                                  → levantar y verificar
Paso 6  Componente tarjeta-bip-card                     → levantar y verificar
Paso 7  Páginas, rutas y layout                         → levantar y probar TODO el CRUD
Paso 8  Tests
```


---

## Paso 0 · Crear el proyecto y levantarlo

**Requisitos:** Node 22+ (se usó Node 24), npm y Angular CLI 21 (`npm i -g @angular/cli@21`).

```powershell
ng new tarjetas-frontend --routing --style=css --skip-git --ssr=false --ai-config=none --defaults
cd tarjetas-frontend
npm start
```

**🎓 Teoría:** Angular 21 genera componentes *standalone* (sin NgModules), **zoneless** (sin zone.js: la pantalla se actualiza con *signals*), pruebas con **Vitest** y archivos sin el sufijo `.component` (`app.ts`, `app.html`). `--routing` crea `app.routes.ts`, donde definiremos las tres rutas de la aplicación.

**✅ Verificar:** abre http://localhost:4200 → página de bienvenida de Angular. Detén con **Ctrl + C**.


---

## Paso 1 · Crear la estructura de carpetas

```powershell
cd src/app
mkdir core/models, core/api, core/interceptors
mkdir shared/ui
mkdir features/tarjetas/components/tarjeta-bip-card
```

**🎓 Teoría — ¿por qué "por feature"?** En proyectos reales conviene agrupar por **funcionalidad** (`features/tarjetas`) en vez de por tipo de archivo. Si mañana agregas "Recargas", creas `features/recargas` sin tocar lo demás. Lo que usan varias features va a `core` (datos y servicios) o `shared` (piezas visuales).

**✅ Verificar:** el árbol de carpetas coincide con el de la introducción.


---

## Paso 2 · Models

**🎓 Teoría — Models.** Son **interfaces de TypeScript**: describen la *forma* de los datos (no tienen lógica). Tener un modelo tipado hace que el editor te avise si escribes `tarjeta.titulo` en vez de `tarjeta.titular`. `Tarjeta` es el **espejo del JSON** que responde el backend (`TarjetaResponse`); `TarjetaRequest` y `TarjetaEdicionRequest` son los cuerpos de POST y PUT (el número no viaja al editar). `REGLAS_TARJETA` repite las reglas del backend para que el formulario avise antes de enviar. `ApiError` es el error ya traducido a un mensaje que la pantalla puede mostrar tal cual.

### 🧩 Archivo 2.1 · `api-error.ts`

**Crear:** en `src/app/core/models` → nuevo archivo `api-error.ts` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/app/core/models/api-error.ts`

```ts
/** Error ya traducido a un mensaje que la interfaz puede mostrar tal cual. */
export class ApiError extends Error {
  constructor(
    message: string,
    readonly status: number,
  ) {
    super(message);
    this.name = 'ApiError';
  }
}
```

### 🧩 Archivo 2.2 · `tarjeta.model.ts`

**Crear:** en `src/app/core/models` → nuevo archivo `tarjeta.model.ts` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/app/core/models/tarjeta.model.ts`

```ts
export type TipoTarjeta = 'NORMAL' | 'TNE_ESCOLAR' | 'TNE_SUPERIOR' | 'ADULTO_MAYOR';

export type EstadoTarjeta = 'ACTIVA' | 'BLOQUEADA';

/** Tarjeta tal como la entrega la API (GET /api/tarjetas). Las fechas llegan como texto ISO. */
export interface Tarjeta {
  id: number;
  numero: string;
  titular: string;
  alias: string | null;
  tipo: TipoTarjeta;
  saldo: number;
  estado: EstadoTarjeta;
  creadaEn: string;
  actualizadaEn: string;
}

/** Datos para registrar una tarjeta (POST /api/tarjetas). El id y el estado los asigna el servidor. */
export interface TarjetaRequest {
  numero: string;
  titular: string;
  alias: string | null;
  tipo: TipoTarjeta;
  saldo: number;
}

/** Datos que se pueden editar (PUT /api/tarjetas/{id}). El número de tarjeta no cambia. */
export interface TarjetaEdicionRequest {
  titular: string;
  alias: string | null;
  tipo: TipoTarjeta;
  saldo: number;
  estado: EstadoTarjeta;
}

/** Reglas que el formulario repite para avisar antes de enviar (el backend las valida igual). */
export const REGLAS_TARJETA = {
  numeroPatron: /^\d{8,10}$/,
  titularMaxLargo: 100,
  aliasMaxLargo: 50,
  saldoMinimo: 0,
  saldoMaximo: 100000,
} as const;
```


#### 🚀 Levantar y ✅ verificar

```powershell
npm start
```

Compila sin errores (todavía no se ve nada nuevo).


---

## Paso 3 · Api, interceptor, proxy y configuración HTTP

**🎓 Teoría — Api.** `TarjetaApi` es el **único** lugar que conoce las rutas del backend (`/api/tarjetas`). Devuelve `Observable`: un "flujo" asíncrono al que el que llama se suscribe.
**Interceptor.** Se ejecuta con **cada** respuesta HTTP: convierte los errores del backend (el JSON `ApiError` que arma `GlobalExceptionHandler`) en un `ApiError` con un mensaje legible, así ningún componente repite `try/catch`.
**Proxy.** El navegador llama a `/api/...` en el **mismo puerto** de Angular (4200) y `proxy.conf.json` lo reenvía al backend (8082). Como el navegador cree que habla con un solo origen, **no hace falta configurar CORS**. Si el backend usa otro puerto, solo cambias el `target` del proxy.

### 🧩 Archivo 3.1 · `tarjeta.api.ts`

**Crear:** en `src/app/core/api` → nuevo archivo `tarjeta.api.ts` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/app/core/api/tarjeta.api.ts`

```ts
import { HttpClient } from '@angular/common/http';
import { Injectable, inject } from '@angular/core';
import { Observable } from 'rxjs';

import { Tarjeta, TarjetaEdicionRequest, TarjetaRequest } from '../models/tarjeta.model';

/** Único lugar que conoce las rutas HTTP del backend (Spring Boot, /api/tarjetas). */
@Injectable({ providedIn: 'root' })
export class TarjetaApi {
  private readonly http = inject(HttpClient);
  private readonly base = '/api/tarjetas';

  listar(): Observable<Tarjeta[]> {
    return this.http.get<Tarjeta[]>(this.base);
  }

  obtener(id: number): Observable<Tarjeta> {
    return this.http.get<Tarjeta>(`${this.base}/${id}`);
  }

  crear(request: TarjetaRequest): Observable<Tarjeta> {
    return this.http.post<Tarjeta>(this.base, request);
  }

  actualizar(id: number, request: TarjetaEdicionRequest): Observable<Tarjeta> {
    return this.http.put<Tarjeta>(`${this.base}/${id}`, request);
  }

  eliminar(id: number): Observable<void> {
    return this.http.delete<void>(`${this.base}/${id}`);
  }
}
```

### 🧩 Archivo 3.2 · `error.interceptor.ts`

**Crear:** en `src/app/core/interceptors` → nuevo archivo `error.interceptor.ts` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/app/core/interceptors/error.interceptor.ts`

```ts
import { HttpErrorResponse, HttpInterceptorFn } from '@angular/common/http';
import { catchError, throwError } from 'rxjs';

import { ApiError } from '../models/api-error';

/** Forma del error que responde el backend (ApiError.java). */
interface ProblemaHttp {
  mensaje?: string;
  detalles?: string[];
}

/** Convierte cualquier error HTTP en un ApiError con un mensaje legible. */
export const errorInterceptor: HttpInterceptorFn = (req, next) =>
  next(req).pipe(
    catchError((error: HttpErrorResponse) =>
      throwError(() => new ApiError(mensajeDe(error), error.status)),
    ),
  );

function mensajeDe(error: HttpErrorResponse): string {
  if (error.status === 0) {
    return 'No hay conexión con el servidor. ¿Está encendido el backend?';
  }
  const problema = error.error as ProblemaHttp | null;
  const detalles = problema?.detalles?.length ? ` (${problema.detalles.join('. ')})` : '';
  return problema?.mensaje ? `${problema.mensaje}${detalles}` : 'Ocurrió un error inesperado';
}
```

### 🧩 Archivo 3.3 · `app.config.ts`

**Crear:** en `src/app` → nuevo archivo `app.config.ts` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/app/app.config.ts`

```ts
import { ApplicationConfig, provideBrowserGlobalErrorListeners } from '@angular/core';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { provideRouter, withComponentInputBinding } from '@angular/router';

import { routes } from './app.routes';
import { errorInterceptor } from './core/interceptors/error.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideBrowserGlobalErrorListeners(),
    // withComponentInputBinding: el :id de la URL llega al componente como input()
    provideRouter(routes, withComponentInputBinding()),
    provideHttpClient(withInterceptors([errorInterceptor])),
  ]
};
```

### 🧩 Archivo 3.4 · `proxy.conf.json`

**Crear:** en `(raíz del proyecto)` → nuevo archivo `proxy.conf.json` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `proxy.conf.json`

```json
{
  "/api": {
    "target": "http://localhost:8082",
    "secure": false
  }
}
```


Activa el proxy en `angular.json`: dentro de `projects → tarjetas-frontend → architect → serve` agrega `"options"` **antes** de `"configurations"`:

```json
"serve": {
  "builder": "@angular/build:dev-server",
  "options": {
    "proxyConfig": "proxy.conf.json"
  },
  "configurations": { ... }
}
```

#### 🚀 Levantar y ✅ verificar

```powershell
npm start
```

Compila sin errores. (Cambiaste `angular.json` y `proxy.conf.json`: si `npm start` ya estaba corriendo, **deténlo y vuelve a levantarlo**.)


---

## Paso 4 · Shared UI y estilos

**🎓 Teoría — Piezas reutilizables.** El **toast** muestra avisos de éxito/error desde cualquier lugar gracias a `ToastService` (un *Singleton*: `providedIn: 'root'`); el **pipe** `clp` transforma `5200` en `$5.200` solo para mostrarlo; `EstadoBadge` dibuja la etiqueta "Activa/Bloqueada" y `ConfirmDialog` es el diálogo de confirmación (los dos reciben datos con `input()` y avisan con `output()`).
**Estilos globales:** `styles.css` define variables de diseño (`--primario`, `--borde`…) y clases reutilizables (`.btn`, `.campo`, `.toast`). Cambiando las variables cambias todo el aspecto.

### 🧩 Archivo 4.1 · `clp.pipe.ts`

**Crear:** en `src/app/shared/ui` → nuevo archivo `clp.pipe.ts` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/app/shared/ui/clp.pipe.ts`

```ts
import { Pipe, PipeTransform } from '@angular/core';

/** Formatea un número como pesos chilenos: 4000 -> $4.000 */
@Pipe({ name: 'clp' })
export class ClpPipe implements PipeTransform {
  private readonly formato = new Intl.NumberFormat('es-CL', {
    style: 'currency',
    currency: 'CLP',
    maximumFractionDigits: 0,
  });

  transform(valor: number | null | undefined): string {
    return valor === null || valor === undefined ? '' : this.formato.format(valor);
  }
}
```

### 🧩 Archivo 4.2 · `toast.service.ts`

**Crear:** en `src/app/shared/ui` → nuevo archivo `toast.service.ts` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/app/shared/ui/toast.service.ts`

```ts
import { Injectable, signal } from '@angular/core';

export interface Aviso {
  id: number;
  tipo: 'exito' | 'error';
  mensaje: string;
}

const DURACION_MS = 4000;

@Injectable({ providedIn: 'root' })
export class ToastService {
  private siguienteId = 0;
  readonly avisos = signal<Aviso[]>([]);

  exito(mensaje: string): void {
    this.mostrar('exito', mensaje);
  }

  error(mensaje: string): void {
    this.mostrar('error', mensaje);
  }

  descartar(id: number): void {
    this.avisos.update((lista) => lista.filter((aviso) => aviso.id !== id));
  }

  private mostrar(tipo: Aviso['tipo'], mensaje: string): void {
    const id = ++this.siguienteId;
    this.avisos.update((lista) => [...lista, { id, tipo, mensaje }]);
    setTimeout(() => this.descartar(id), DURACION_MS);
  }
}
```

### 🧩 Archivo 4.3 · `toast.ts`

**Crear:** en `src/app/shared/ui` → nuevo archivo `toast.ts` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/app/shared/ui/toast.ts`

```ts
import { Component, inject } from '@angular/core';

import { ToastService } from './toast.service';

@Component({
  selector: 'app-toast',
  template: `
    <div class="toasts" aria-live="polite">
      @for (aviso of toast.avisos(); track aviso.id) {
        <div class="toast" [class.toast-error]="aviso.tipo === 'error'">
          <span>{{ aviso.mensaje }}</span>
          <button type="button" class="toast-cerrar" aria-label="Cerrar aviso" (click)="toast.descartar(aviso.id)">✕</button>
        </div>
      }
    </div>
  `,
})
export class Toast {
  protected readonly toast = inject(ToastService);
}
```

### 🧩 Archivo 4.4 · `estado-badge.ts`

**Crear:** en `src/app/shared/ui` → nuevo archivo `estado-badge.ts` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/app/shared/ui/estado-badge.ts`

```ts
import { ChangeDetectionStrategy, Component, input } from '@angular/core';
import { EstadoTarjeta } from '../../core/models/tarjeta.model';

/** Componente "tonto" (presentacional): solo recibe datos por input y los dibuja. */
@Component({
  selector: 'app-estado-badge',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <span class="badge" [class.badge-ok]="estado() === 'ACTIVA'" [class.badge-off]="estado() === 'BLOQUEADA'">
      {{ estado() === 'ACTIVA' ? 'Activa' : 'Bloqueada' }}
    </span>
  `,
  styles: `
    .badge { display: inline-block; padding: .15rem .6rem; border-radius: 999px; font-size: .75rem; font-weight: 600; }
    .badge-ok { background: #dcfce7; color: #166534; }
    .badge-off { background: #fee2e2; color: #991b1b; }
  `,
})
export class EstadoBadge {
  readonly estado = input.required<EstadoTarjeta>();
}
```

### 🧩 Archivo 4.5 · `confirm-dialog.ts`

**Crear:** en `src/app/shared/ui` → nuevo archivo `confirm-dialog.ts` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/app/shared/ui/confirm-dialog.ts`

```ts
import { ChangeDetectionStrategy, Component, input, output } from '@angular/core';

/** Diálogo de confirmación reutilizable. El padre decide cuándo se muestra y qué hacer. */
@Component({
  selector: 'app-confirm-dialog',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="fondo" (click)="cancelar.emit()">
      <div class="dialogo" role="alertdialog" aria-modal="true" [attr.aria-label]="titulo()" (click)="$event.stopPropagation()">
        <h2>{{ titulo() }}</h2>
        <p>{{ mensaje() }}</p>
        <div class="acciones">
          <button type="button" class="btn" (click)="cancelar.emit()">Cancelar</button>
          <button type="button" class="btn btn-peligro" (click)="confirmar.emit()">{{ textoConfirmar() }}</button>
        </div>
      </div>
    </div>
  `,
  styles: `
    .fondo { position: fixed; inset: 0; background: rgb(15 23 42 / .55); display: grid; place-items: center; z-index: 40; padding: 1rem; }
    .dialogo { background: var(--superficie); border-radius: .9rem; padding: 1.5rem; width: min(26rem, 100%); box-shadow: 0 20px 50px rgb(0 0 0 / .3); }
    h2 { margin: 0 0 .5rem; font-size: 1.15rem; }
    p { margin: 0 0 1.25rem; color: var(--texto-suave); }
    .acciones { display: flex; justify-content: flex-end; gap: .5rem; }
  `,
})
export class ConfirmDialog {
  readonly titulo = input.required<string>();
  readonly mensaje = input.required<string>();
  readonly textoConfirmar = input('Eliminar');
  readonly confirmar = output<void>();
  readonly cancelar = output<void>();
}
```

### 🧩 Archivo 4.6 · `styles.css`

**Crear:** en `src` → nuevo archivo `styles.css` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/styles.css`

```css
/* Estilos globales: variables de diseño + clases reutilizables (botones, campos y avisos). */
:root {
  --primario: #e11d48;
  --primario-oscuro: #be123c;
  --oscuro: #0f172a;
  --fondo: #f1f5f9;
  --superficie: #ffffff;
  --borde: #e2e8f0;
  --texto: #0f172a;
  --texto-suave: #64748b;
}

* { box-sizing: border-box; }

body {
  margin: 0;
  font-family: system-ui, -apple-system, 'Segoe UI', Roboto, sans-serif;
  background: var(--fondo);
  color: var(--texto);
  line-height: 1.5;
}

.btn {
  display: inline-flex; align-items: center; justify-content: center;
  padding: .55rem 1rem; border-radius: .6rem; border: 1px solid var(--borde);
  background: var(--superficie); color: var(--texto);
  font: inherit; font-weight: 600; font-size: .9rem; text-decoration: none; cursor: pointer;
}
.btn:hover { background: #f8fafc; }
.btn:focus-visible, .campo:focus-visible { outline: 2px solid var(--primario); outline-offset: 2px; }
.btn:disabled { opacity: .6; cursor: not-allowed; }
.btn-primario { background: var(--primario); border-color: var(--primario); color: #fff; }
.btn-primario:hover { background: var(--primario-oscuro); }
.btn-peligro { background: #dc2626; border-color: #dc2626; color: #fff; }
.btn-peligro:hover { background: #b91c1c; }
.btn-peligro-suave { color: #b91c1c; }
.btn-peligro-suave:hover { background: #fef2f2; }
.btn-chico { padding: .3rem .7rem; font-size: .8rem; }

.campo {
  width: 100%; padding: .55rem .75rem; border-radius: .6rem; border: 1px solid #cbd5e1;
  background: var(--superficie); color: var(--texto); font: inherit;
}
.campo:disabled { background: #f1f5f9; color: var(--texto-suave); }
.campo.ng-invalid.ng-touched { border-color: #dc2626; }

/* ---- Avisos (toast) ---- */
.toasts {
  position: fixed; right: 1rem; bottom: 1rem; z-index: 60;
  display: grid; gap: .5rem; max-width: min(24rem, calc(100vw - 2rem));
}
.toast {
  display: flex; align-items: center; justify-content: space-between; gap: .75rem;
  padding: .75rem 1rem; border-radius: .6rem;
  background: #14532d; color: #fff; box-shadow: 0 8px 24px rgb(0 0 0 / .25);
}
.toast-error { background: #7f1d1d; }
.toast-cerrar { background: none; border: 0; color: inherit; font-size: 1rem; cursor: pointer; line-height: 1; }
```


#### 🚀 Levantar y ✅ verificar

```powershell
npm start
```

Compila sin errores. El fondo de la página de bienvenida ya cambia de color por los estilos globales.


---

## Paso 5 · Store y textos

**🎓 Teoría — Store (patrón Facade + Observer).** El store guarda el **estado en signals** (`tarjetas`, `busqueda`, `filtroEstado`, `cargando`, `error`) y expone **computed** (`tarjetasFiltradas`, `resumen`) que se **recalculan solos** cuando cambia algo de lo que dependen: nadie actualiza la pantalla a mano. Las operaciones (`crear`, `actualizar`, `eliminar`) devuelven un `Observable` que **emite solo si salió bien**; en caso de error el store muestra el aviso y corta el flujo (`EMPTY`), así la página navega o cierra el diálogo únicamente si todo salió bien.

### 🧩 Archivo 5.1 · `tarjeta-labels.ts`

**Crear:** en `src/app/features/tarjetas` → nuevo archivo `tarjeta-labels.ts` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/app/features/tarjetas/tarjeta-labels.ts`

```ts
import { EstadoTarjeta, TipoTarjeta } from '../../core/models/tarjeta.model';

/** Textos para mostrar en pantalla (son cosas de la vista, por eso no están en el modelo). */
export const TIPOS_TARJETA: ReadonlyArray<{ valor: TipoTarjeta; texto: string }> = [
  { valor: 'NORMAL', texto: 'Normal' },
  { valor: 'TNE_ESCOLAR', texto: 'TNE Escolar' },
  { valor: 'TNE_SUPERIOR', texto: 'TNE Educación Superior' },
  { valor: 'ADULTO_MAYOR', texto: 'Adulto Mayor' },
];

export const ESTADOS_TARJETA: ReadonlyArray<{ valor: EstadoTarjeta; texto: string }> = [
  { valor: 'ACTIVA', texto: 'Activa' },
  { valor: 'BLOQUEADA', texto: 'Bloqueada' },
];

export function textoTipo(tipo: TipoTarjeta): string {
  return TIPOS_TARJETA.find((t) => t.valor === tipo)?.texto ?? tipo;
}
```

### 🧩 Archivo 5.2 · `tarjetas.store.ts`

**Crear:** en `src/app/features/tarjetas` → nuevo archivo `tarjetas.store.ts` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/app/features/tarjetas/tarjetas.store.ts`

```ts
import { Injectable, computed, inject, signal } from '@angular/core';
import { EMPTY, Observable, catchError, finalize, map, switchMap, tap } from 'rxjs';

import { TarjetaApi } from '../../core/api/tarjeta.api';
import { ApiError } from '../../core/models/api-error';
import {
  EstadoTarjeta,
  Tarjeta,
  TarjetaEdicionRequest,
  TarjetaRequest,
} from '../../core/models/tarjeta.model';
import { ToastService } from '../../shared/ui/toast.service';

export type FiltroEstado = 'TODAS' | EstadoTarjeta;

/**
 * Estado de las pantallas (patrón Store/Facade con signals).
 * Las páginas leen signals y llaman a métodos; nunca hablan con la API directamente.
 */
@Injectable({ providedIn: 'root' })
export class TarjetasStore {
  private readonly api = inject(TarjetaApi);
  private readonly toast = inject(ToastService);

  /** Todas las tarjetas que hay en el servidor. */
  readonly tarjetas = signal<Tarjeta[]>([]);
  readonly cargando = signal(false);
  readonly error = signal<string | null>(null);
  readonly busqueda = signal('');
  readonly filtroEstado = signal<FiltroEstado>('TODAS');

  /** Las tarjetas que se ven según el texto buscado y el estado elegido. */
  readonly tarjetasFiltradas = computed(() => {
    const texto = this.busqueda().trim().toLowerCase();
    const estado = this.filtroEstado();
    return this.tarjetas().filter((t) => {
      const coincideEstado = estado === 'TODAS' || t.estado === estado;
      const coincideTexto =
        !texto ||
        t.numero.includes(texto) ||
        t.titular.toLowerCase().includes(texto) ||
        (t.alias ?? '').toLowerCase().includes(texto);
      return coincideEstado && coincideTexto;
    });
  });

  /** Datos de las tres cajas de resumen. */
  readonly resumen = computed(() => {
    const todas = this.tarjetas();
    return {
      total: todas.length,
      activas: todas.filter((t) => t.estado === 'ACTIVA').length,
      saldoTotal: todas.reduce((suma, t) => suma + t.saldo, 0),
    };
  });

  cargar(): void {
    this.cargando.set(true);
    this.error.set(null);
    this.refrescar()
      .pipe(finalize(() => this.cargando.set(false)))
      .subscribe({ error: (e: ApiError) => this.error.set(e.message) });
  }

  buscar(texto: string): void {
    this.busqueda.set(texto);
  }

  filtrarPorEstado(estado: FiltroEstado): void {
    this.filtroEstado.set(estado);
  }

  /** Para la pantalla de edición: trae una tarjeta por id. */
  obtener(id: number): Observable<Tarjeta> {
    return this.api.obtener(id);
  }

  /**
   * Cada operación devuelve un Observable que emite solo si salió bien;
   * así la pantalla navega o cierra el diálogo únicamente en caso de éxito.
   */
  crear(request: TarjetaRequest): Observable<void> {
    return this.api.crear(request).pipe(
      switchMap(() => this.refrescar()),
      tap(() => this.toast.exito('Tarjeta registrada')),
      catchError(this.fallar),
    );
  }

  actualizar(id: number, request: TarjetaEdicionRequest): Observable<void> {
    return this.api.actualizar(id, request).pipe(
      switchMap(() => this.refrescar()),
      tap(() => this.toast.exito('Tarjeta actualizada')),
      catchError(this.fallar),
    );
  }

  eliminar(id: number): Observable<void> {
    return this.api.eliminar(id).pipe(
      switchMap(() => this.refrescar()),
      tap(() => this.toast.exito('Tarjeta eliminada')),
      catchError(this.fallar),
    );
  }

  /** Vuelve a pedir la lista al servidor. */
  private refrescar(): Observable<void> {
    return this.api.listar().pipe(
      tap((tarjetas) => this.tarjetas.set(tarjetas)),
      map(() => undefined),
    );
  }

  /** Muestra el error y corta el flujo: el Observable termina sin emitir. */
  private readonly fallar = (error: ApiError): Observable<never> => {
    this.toast.error(error.message);
    return EMPTY;
  };
}
```


#### 🚀 Levantar y ✅ verificar

```powershell
npm start
```

Compila sin errores. Aún no hay pantalla que use el store.


---

## Paso 6 · Componente tarjeta-bip-card

**🎓 Teoría — Componente "tonto".** `TarjetaBipCard` dibuja **una** tarjeta con aspecto de Bip! (degradado rojo, gris si está bloqueada). Recibe la tarjeta con `input()` y avisa con `output()` cuando se pulsa **Eliminar**; **no** conoce el store ni el backend, por eso se reutiliza y prueba fácilmente. Los valores calculados (`numeroFormateado`, `tipoTexto`) son `computed`.

### 🧩 Archivo 6.1 · `tarjeta-bip-card.ts`

**Crear:** en `src/app/features/tarjetas/components/tarjeta-bip-card` → nuevo archivo `tarjeta-bip-card.ts` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/app/features/tarjetas/components/tarjeta-bip-card/tarjeta-bip-card.ts`

```ts
import { ChangeDetectionStrategy, Component, computed, input, output } from '@angular/core';
import { RouterLink } from '@angular/router';

import { Tarjeta } from '../../../../core/models/tarjeta.model';
import { ClpPipe } from '../../../../shared/ui/clp.pipe';
import { EstadoBadge } from '../../../../shared/ui/estado-badge';
import { textoTipo } from '../../tarjeta-labels';

/** Componente de presentación: dibuja UNA tarjeta y avisa al padre con outputs (no llama al backend). */
@Component({
  selector: 'app-tarjeta-bip-card',
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [RouterLink, ClpPipe, EstadoBadge],
  templateUrl: './tarjeta-bip-card.html',
  styleUrl: './tarjeta-bip-card.css',
})
export class TarjetaBipCard {
  readonly tarjeta = input.required<Tarjeta>();
  readonly eliminar = output<Tarjeta>();

  protected readonly numeroFormateado = computed(() =>
    this.tarjeta().numero.replace(/(\d{4})(?=\d)/g, '$1 '),
  );
  protected readonly tipoTexto = computed(() => textoTipo(this.tarjeta().tipo));
}
```

### 🧩 Archivo 6.2 · `tarjeta-bip-card.html`

**Crear:** en `src/app/features/tarjetas/components/tarjeta-bip-card` → nuevo archivo `tarjeta-bip-card.html` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/app/features/tarjetas/components/tarjeta-bip-card/tarjeta-bip-card.html`

```html
<article class="tarjeta" [class.bloqueada]="tarjeta().estado === 'BLOQUEADA'">
  <div class="frente">
    <div class="fila">
      <span class="logo">bip!</span>
      <span class="tipo">{{ tipoTexto() }}</span>
    </div>
    <div class="numero">{{ numeroFormateado() }}</div>
    <div class="fila abajo">
      <div>
        <div class="titular">{{ tarjeta().titular }}</div>
        @if (tarjeta().alias) {
          <div class="alias">{{ tarjeta().alias }}</div>
        }
      </div>
      <div class="saldo">
        <span class="saldo-etiqueta">Saldo</span>
        {{ tarjeta().saldo | clp }}
      </div>
    </div>
  </div>
  <footer class="pie">
    <app-estado-badge [estado]="tarjeta().estado" />
    <div class="acciones">
      <a class="btn btn-chico" [routerLink]="['/tarjetas', tarjeta().id, 'editar']">Editar</a>
      <button type="button" class="btn btn-chico btn-peligro-suave" (click)="eliminar.emit(tarjeta())">Eliminar</button>
    </div>
  </footer>
</article>
```

### 🧩 Archivo 6.3 · `tarjeta-bip-card.css`

**Crear:** en `src/app/features/tarjetas/components/tarjeta-bip-card` → nuevo archivo `tarjeta-bip-card.css` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/app/features/tarjetas/components/tarjeta-bip-card/tarjeta-bip-card.css`

```css
:host { display: block; }

.tarjeta {
  background: var(--superficie);
  border-radius: 1rem;
  overflow: hidden;
  box-shadow: 0 2px 10px rgb(15 23 42 / .08);
  border: 1px solid var(--borde);
}

.frente {
  padding: 1.1rem 1.25rem 1rem;
  color: #fff;
  background: linear-gradient(135deg, #e11d48 0%, #be123c 55%, #881337 100%);
  display: grid;
  gap: 1rem;
}

.bloqueada .frente {
  background: linear-gradient(135deg, #64748b 0%, #475569 60%, #334155 100%);
}

.fila { display: flex; justify-content: space-between; align-items: center; gap: .75rem; }
.fila.abajo { align-items: flex-end; }

.logo { font-weight: 800; font-size: 1.5rem; letter-spacing: -.03em; font-style: italic; }
.tipo { font-size: .72rem; text-transform: uppercase; letter-spacing: .08em; background: rgb(255 255 255 / .2); padding: .2rem .6rem; border-radius: 999px; }

.numero { font-size: 1.35rem; font-weight: 600; letter-spacing: .12em; font-variant-numeric: tabular-nums; }

.titular { font-weight: 600; }
.alias { font-size: .8rem; opacity: .85; }

.saldo { text-align: right; font-size: 1.25rem; font-weight: 700; display: grid; }
.saldo-etiqueta { font-size: .65rem; font-weight: 500; text-transform: uppercase; letter-spacing: .08em; opacity: .85; }

.pie { display: flex; justify-content: space-between; align-items: center; padding: .7rem 1rem; }
.acciones { display: flex; gap: .4rem; }
```


#### 🚀 Levantar y ✅ verificar

```powershell
npm start
```

Compila sin errores. Todavía no se ve porque falta la página que lo usa.


---

## Paso 7 · Páginas, rutas y layout (todo se conecta)

**🎓 Teoría — Página = "Controller" de la vista.** `TarjetasPage` (listado) y `TarjetaFormPage` (formulario) son los **únicos** componentes que conocen al store. El listado carga los datos al iniciar (`ngOnInit`), muestra el resumen, el buscador y el filtro, y pide confirmación antes de eliminar. El formulario sirve para **crear** (`/tarjetas/nueva`) y **editar** (`/tarjetas/:id/editar`): el parámetro `:id` llega como `input()` gracias a `withComponentInputBinding()`; en edición el número se deshabilita porque no se puede cambiar. Usa **formularios reactivos** con los mismos validadores que el backend.
**Rutas:** cada pantalla se carga con *lazy loading* (`loadComponent`): se descarga solo cuando se visita.

### 🧩 Archivo 7.1 · `tarjetas.page.ts`

**Crear:** en `src/app/features/tarjetas` → nuevo archivo `tarjetas.page.ts` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/app/features/tarjetas/tarjetas.page.ts`

```ts
import { ChangeDetectionStrategy, Component, OnInit, inject, signal } from '@angular/core';
import { RouterLink } from '@angular/router';

import { Tarjeta } from '../../core/models/tarjeta.model';
import { ClpPipe } from '../../shared/ui/clp.pipe';
import { ConfirmDialog } from '../../shared/ui/confirm-dialog';
import { TarjetaBipCard } from './components/tarjeta-bip-card/tarjeta-bip-card';
import { FiltroEstado, TarjetasStore } from './tarjetas.store';

/**
 * Página del listado = "Controller" de la vista (MVC): es la única que conoce al store.
 * Las tarjetas las dibuja un componente presentacional (TarjetaBipCard).
 */
@Component({
  selector: 'app-tarjetas-page',
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [RouterLink, ClpPipe, TarjetaBipCard, ConfirmDialog],
  templateUrl: './tarjetas.page.html',
  styleUrl: './tarjetas.page.css',
})
export class TarjetasPage implements OnInit {
  protected readonly store = inject(TarjetasStore);

  /** Tarjeta que espera confirmación para eliminarse. */
  protected readonly porEliminar = signal<Tarjeta | null>(null);

  ngOnInit(): void {
    this.store.cargar();
  }

  protected buscar(evento: Event): void {
    this.store.buscar((evento.target as HTMLInputElement).value);
  }

  protected filtrar(evento: Event): void {
    this.store.filtrarPorEstado((evento.target as HTMLSelectElement).value as FiltroEstado);
  }

  protected pedirConfirmacion(tarjeta: Tarjeta): void {
    this.porEliminar.set(tarjeta);
  }

  protected cancelarEliminacion(): void {
    this.porEliminar.set(null);
  }

  protected confirmarEliminacion(): void {
    const tarjeta = this.porEliminar();
    if (!tarjeta) return;
    this.porEliminar.set(null);
    this.store.eliminar(tarjeta.id).subscribe();
  }
}
```

### 🧩 Archivo 7.2 · `tarjetas.page.html`

**Crear:** en `src/app/features/tarjetas` → nuevo archivo `tarjetas.page.html` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/app/features/tarjetas/tarjetas.page.html`

```html
<header class="encabezado">
  <div>
    <h1>Mis tarjetas Bip!</h1>
    <p class="subtitulo">Registra, edita y elimina tus tarjetas.</p>
  </div>
  <a class="btn btn-primario" routerLink="/tarjetas/nueva">+ Nueva tarjeta</a>
</header>

<section class="resumen" aria-label="Resumen">
  <div class="dato">
    <span class="dato-valor">{{ store.resumen().total }}</span>
    <span class="dato-texto">Tarjetas</span>
  </div>
  <div class="dato">
    <span class="dato-valor">{{ store.resumen().activas }}</span>
    <span class="dato-texto">Activas</span>
  </div>
  <div class="dato">
    <span class="dato-valor">{{ store.resumen().saldoTotal | clp }}</span>
    <span class="dato-texto">Saldo total</span>
  </div>
</section>

<section class="filtros">
  <input
    class="campo"
    type="search"
    placeholder="Buscar por número, titular o alias"
    aria-label="Buscar tarjetas"
    [value]="store.busqueda()"
    (input)="buscar($event)"
  />
  <select class="campo" aria-label="Filtrar por estado" [value]="store.filtroEstado()" (change)="filtrar($event)">
    <option value="TODAS">Todos los estados</option>
    <option value="ACTIVA">Activas</option>
    <option value="BLOQUEADA">Bloqueadas</option>
  </select>
</section>

@if (store.cargando()) {
  <p class="mensaje">Cargando tarjetas…</p>
} @else if (store.error(); as mensajeError) {
  <div class="mensaje mensaje-error">
    <p>{{ mensajeError }}</p>
    <button type="button" class="btn" (click)="store.cargar()">Reintentar</button>
  </div>
} @else if (store.tarjetas().length === 0) {
  <div class="mensaje">
    <p>Aún no tienes tarjetas registradas.</p>
    <a class="btn btn-primario" routerLink="/tarjetas/nueva">Registrar la primera</a>
  </div>
} @else if (store.tarjetasFiltradas().length === 0) {
  <p class="mensaje">Ninguna tarjeta coincide con la búsqueda.</p>
} @else {
  <section class="grilla">
    @for (tarjeta of store.tarjetasFiltradas(); track tarjeta.id) {
      <app-tarjeta-bip-card [tarjeta]="tarjeta" (eliminar)="pedirConfirmacion($event)" />
    }
  </section>
}

@if (porEliminar(); as tarjeta) {
  <app-confirm-dialog
    titulo="Eliminar tarjeta"
    [mensaje]="'¿Seguro que quieres eliminar la tarjeta ' + tarjeta.numero + ' de ' + tarjeta.titular + '? Esta acción no se puede deshacer.'"
    (confirmar)="confirmarEliminacion()"
    (cancelar)="cancelarEliminacion()"
  />
}
```

### 🧩 Archivo 7.3 · `tarjetas.page.css`

**Crear:** en `src/app/features/tarjetas` → nuevo archivo `tarjetas.page.css` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/app/features/tarjetas/tarjetas.page.css`

```css
.encabezado { display: flex; justify-content: space-between; align-items: flex-start; gap: 1rem; flex-wrap: wrap; margin-bottom: .9rem; }
h1 { margin: 0; font-size: 1.6rem; }
.subtitulo { margin: .2rem 0 0; color: var(--texto-suave); }

.resumen { display: grid; grid-template-columns: repeat(auto-fit, minmax(9rem, 1fr)); gap: .75rem; margin-bottom: .9rem; }
.dato { background: var(--superficie); border: 1px solid var(--borde); border-radius: .8rem; padding: .8rem 1rem; display: grid; }
.dato-valor { font-size: 1.4rem; font-weight: 700; }
.dato-texto { font-size: .8rem; color: var(--texto-suave); }

/* Buscador y filtro en UNA fila (se apilan solo en pantallas muy angostas) */
.filtros { display: flex; gap: .6rem; flex-wrap: wrap; margin-bottom: .9rem; }
.filtros input { flex: 1 1 14rem; }
.filtros select { flex: 0 0 13rem; width: 13rem; }

/* 3 tarjetas por fila en pantallas normales; 2 o 1 si el ancho no alcanza. Nunca desborda hacia los lados. */
.grilla { display: grid; grid-template-columns: repeat(auto-fit, minmax(min(100%, 15rem), 1fr)); gap: .9rem; }

.mensaje { background: var(--superficie); border: 1px dashed var(--borde); border-radius: .8rem; padding: 2rem 1rem; text-align: center; color: var(--texto-suave); display: grid; gap: .75rem; justify-items: center; }
.mensaje p { margin: 0; }
.mensaje-error { border-color: #fca5a5; color: #991b1b; }
```

### 🧩 Archivo 7.4 · `tarjeta-form.page.ts`

**Crear:** en `src/app/features/tarjetas` → nuevo archivo `tarjeta-form.page.ts` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/app/features/tarjetas/tarjeta-form.page.ts`

```ts
import { ChangeDetectionStrategy, Component, OnInit, computed, inject, input, signal } from '@angular/core';
import { NonNullableFormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';
import { Router, RouterLink } from '@angular/router';

import { EstadoTarjeta, REGLAS_TARJETA, TipoTarjeta } from '../../core/models/tarjeta.model';
import { ESTADOS_TARJETA, TIPOS_TARJETA } from './tarjeta-labels';
import { TarjetasStore } from './tarjetas.store';

/**
 * Pantalla de formulario reutilizada para CREAR (/tarjetas/nueva) y EDITAR (/tarjetas/:id/editar).
 * El router entrega el parámetro :id como input gracias a withComponentInputBinding().
 */
@Component({
  selector: 'app-tarjeta-form-page',
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [ReactiveFormsModule, RouterLink],
  templateUrl: './tarjeta-form.page.html',
  styleUrl: './tarjeta-form.page.css',
})
export class TarjetaFormPage implements OnInit {
  private readonly fb = inject(NonNullableFormBuilder);
  private readonly store = inject(TarjetasStore);
  private readonly router = inject(Router);

  /** Viene de la URL. Si no existe, estamos creando. */
  readonly id = input<string>();

  protected readonly tipos = TIPOS_TARJETA;
  protected readonly estados = ESTADOS_TARJETA;
  protected readonly reglas = REGLAS_TARJETA;

  protected readonly esEdicion = computed(() => this.id() !== undefined);
  protected readonly cargando = signal(false);
  protected readonly guardando = signal(false);
  protected readonly errorCarga = signal<string | null>(null);
  protected readonly erroresServidor = signal<string[]>([]);

  protected readonly form = this.fb.group({
    numero: ['', [Validators.required, Validators.pattern(REGLAS_TARJETA.numeroPatron)]],
    titular: ['', [Validators.required, Validators.maxLength(REGLAS_TARJETA.titularMaxLargo)]],
    alias: ['', [Validators.maxLength(REGLAS_TARJETA.aliasMaxLargo)]],
    tipo: ['NORMAL' as TipoTarjeta, [Validators.required]],
    saldo: [
      0,
      [
        Validators.required,
        Validators.min(REGLAS_TARJETA.saldoMinimo),
        Validators.max(REGLAS_TARJETA.saldoMaximo),
      ],
    ],
    estado: ['ACTIVA' as EstadoTarjeta, [Validators.required]],
  });

  ngOnInit(): void {
    const id = this.id();
    if (id === undefined) return;

    // En edición el número no se puede cambiar.
    this.form.controls.numero.disable();
    this.cargando.set(true);
    this.store.obtener(Number(id)).subscribe({
      next: (tarjeta) => {
        this.form.patchValue({ ...tarjeta, alias: tarjeta.alias ?? '' });
        this.cargando.set(false);
      },
      error: (e: Error) => {
        this.errorCarga.set(e.message);
        this.cargando.set(false);
      },
    });
  }

  /** ¿El campo es inválido y el usuario ya lo tocó? */
  protected invalido(nombre: 'numero' | 'titular' | 'alias' | 'tipo' | 'saldo' | 'estado'): boolean {
    const control = this.form.controls[nombre];
    return control.invalid && (control.touched || control.dirty);
  }

  protected guardar(): void {
    if (this.form.invalid) {
      this.form.markAllAsTouched();
      return;
    }
    this.guardando.set(true);
    this.erroresServidor.set([]);

    const v = this.form.getRawValue();
    const alias = v.alias.trim() || null;
    const id = this.id();

    const operacion =
      id === undefined
        ? this.store.crear({ numero: v.numero.trim(), titular: v.titular.trim(), alias, tipo: v.tipo, saldo: v.saldo })
        : this.store.actualizar(Number(id), {
            titular: v.titular.trim(),
            alias,
            tipo: v.tipo,
            saldo: v.saldo,
            estado: v.estado,
          });

    // Si el servidor rechaza, el store ya mostró el aviso y no emite: se reactiva el botón.
    operacion.subscribe({
      next: () => void this.router.navigate(['/tarjetas']),
      complete: () => this.guardando.set(false),
    });
  }
}
```

### 🧩 Archivo 7.5 · `tarjeta-form.page.html`

**Crear:** en `src/app/features/tarjetas` → nuevo archivo `tarjeta-form.page.html` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/app/features/tarjetas/tarjeta-form.page.html`

```html
<header class="encabezado">
  <a class="volver" routerLink="/tarjetas">← Volver</a>
  <h1>{{ esEdicion() ? 'Editar tarjeta' : 'Nueva tarjeta' }}</h1>
</header>

@if (cargando()) {
  <p class="aviso">Cargando datos de la tarjeta…</p>
} @else if (errorCarga(); as mensaje) {
  <div class="aviso aviso-error">
    <p>{{ mensaje }}</p>
    <a class="btn" routerLink="/tarjetas">Volver al listado</a>
  </div>
} @else {
  <form class="formulario" [formGroup]="form" (ngSubmit)="guardar()" novalidate>
    @if (erroresServidor().length > 0) {
      <div class="aviso aviso-error" role="alert">
        @for (texto of erroresServidor(); track $index) {
          <p>{{ texto }}</p>
        }
      </div>
    }

    <div class="grupo">
      <label for="numero">Número de tarjeta</label>
      <input id="numero" class="campo" formControlName="numero" inputmode="numeric" maxlength="10" placeholder="Ej: 12345678" />
      @if (esEdicion()) {
        <small class="ayuda">El número no se puede modificar.</small>
      } @else if (invalido('numero')) {
        <small class="error">Ingresa entre 8 y 10 dígitos.</small>
      }
    </div>

    <div class="grupo">
      <label for="titular">Titular</label>
      <input id="titular" class="campo" formControlName="titular" placeholder="Nombre del titular" />
      @if (invalido('titular')) {
        <small class="error">El titular es obligatorio (máximo {{ reglas.titularMaxLargo }} caracteres).</small>
      }
    </div>

    <div class="grupo">
      <label for="alias">Alias (opcional)</label>
      <input id="alias" class="campo" formControlName="alias" placeholder="Ej: Mi Bip del trabajo" />
      @if (invalido('alias')) {
        <small class="error">Máximo {{ reglas.aliasMaxLargo }} caracteres.</small>
      }
    </div>

    <div class="dos-columnas">
      <div class="grupo">
        <label for="tipo">Tipo de tarjeta</label>
        <select id="tipo" class="campo" formControlName="tipo">
          @for (tipo of tipos; track tipo.valor) {
            <option [value]="tipo.valor">{{ tipo.texto }}</option>
          }
        </select>
      </div>

      <div class="grupo">
        <label for="saldo">Saldo (CLP)</label>
        <input id="saldo" class="campo" type="number" formControlName="saldo" min="0" step="100" />
        @if (invalido('saldo')) {
          <small class="error">El saldo debe estar entre $0 y $100.000.</small>
        }
      </div>
    </div>

    @if (esEdicion()) {
      <div class="grupo">
        <label for="estado">Estado</label>
        <select id="estado" class="campo" formControlName="estado">
          @for (estado of estados; track estado.valor) {
            <option [value]="estado.valor">{{ estado.texto }}</option>
          }
        </select>
      </div>
    }

    <div class="acciones">
      <a class="btn" routerLink="/tarjetas">Cancelar</a>
      <button type="submit" class="btn btn-primario" [disabled]="guardando()">
        {{ guardando() ? 'Guardando…' : esEdicion() ? 'Guardar cambios' : 'Registrar tarjeta' }}
      </button>
    </div>
  </form>
}
```

### 🧩 Archivo 7.6 · `tarjeta-form.page.css`

**Crear:** en `src/app/features/tarjetas` → nuevo archivo `tarjeta-form.page.css` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/app/features/tarjetas/tarjeta-form.page.css`

```css
:host { display: block; max-width: 36rem; margin: 0 auto; }
.encabezado { margin-bottom: 1.25rem; }
h1 { margin: .25rem 0 0; font-size: 1.6rem; }
.volver { color: var(--texto-suave); text-decoration: none; font-size: .9rem; }
.volver:hover { color: var(--primario); }

.formulario { background: var(--superficie); border: 1px solid var(--borde); border-radius: 1rem; padding: 1.5rem; display: grid; gap: 1rem; }
.grupo { display: grid; gap: .3rem; }
label { font-size: .85rem; font-weight: 600; }
.dos-columnas { display: grid; grid-template-columns: 1fr 1fr; gap: 1rem; }
.ayuda { color: var(--texto-suave); }
.error { color: #b91c1c; }
.acciones { display: flex; justify-content: flex-end; gap: .5rem; margin-top: .5rem; }

.aviso { background: var(--superficie); border: 1px dashed var(--borde); border-radius: .8rem; padding: 1rem; color: var(--texto-suave); }
.aviso p { margin: 0 0 .25rem; }
.aviso-error { border-color: #fca5a5; color: #991b1b; background: #fef2f2; }

@media (max-width: 30rem) {
  .dos-columnas { grid-template-columns: 1fr; }
}
```

### 🧩 Archivo 7.7 · `app.routes.ts`

**Crear:** en `src/app` → nuevo archivo `app.routes.ts` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/app/app.routes.ts`

```ts
import { Routes } from '@angular/router';

/** Cada pantalla se carga "perezosamente" (lazy loading): solo se descarga cuando se visita. */
export const routes: Routes = [
  { path: '', pathMatch: 'full', redirectTo: 'tarjetas' },
  {
    path: 'tarjetas',
    title: 'Mis tarjetas Bip!',
    loadComponent: () => import('./features/tarjetas/tarjetas.page').then((m) => m.TarjetasPage),
  },
  {
    path: 'tarjetas/nueva',
    title: 'Nueva tarjeta',
    loadComponent: () => import('./features/tarjetas/tarjeta-form.page').then((m) => m.TarjetaFormPage),
  },
  {
    path: 'tarjetas/:id/editar',
    title: 'Editar tarjeta',
    loadComponent: () => import('./features/tarjetas/tarjeta-form.page').then((m) => m.TarjetaFormPage),
  },
  { path: '**', redirectTo: 'tarjetas' },
];
```

### 🧩 Archivo 7.8 · `app.ts`

**Crear:** en `src/app` → nuevo archivo `app.ts` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/app/app.ts`

```ts
import { ChangeDetectionStrategy, Component } from '@angular/core';
import { RouterLink, RouterOutlet } from '@angular/router';

import { Toast } from './shared/ui/toast';

/** Layout principal: cabecera + zona donde el router dibuja cada pantalla. */
@Component({
  selector: 'app-root',
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [RouterOutlet, RouterLink, Toast],
  templateUrl: './app.html',
  styleUrl: './app.css',
})
export class App {}
```

### 🧩 Archivo 7.9 · `app.html`

**Crear:** en `src/app` → nuevo archivo `app.html` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/app/app.html`

```html
<header class="barra">
  <a class="marca" routerLink="/tarjetas"><span class="marca-logo">bip!</span> Gestor de tarjetas</a>
</header>

<main class="contenido">
  <router-outlet />
</main>

<app-toast />
```

### 🧩 Archivo 7.10 · `app.css`

**Crear:** en `src/app` → nuevo archivo `app.css` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/app/app.css`

```css
.barra { background: var(--oscuro); color: #fff; padding: .8rem 1rem; }
.marca { color: inherit; text-decoration: none; font-weight: 600; display: inline-flex; align-items: center; gap: .6rem; max-width: 70rem; margin: 0 auto; }
.marca-logo { background: var(--primario); padding: .1rem .55rem; border-radius: .4rem; font-style: italic; font-weight: 800; letter-spacing: -.03em; }
.barra { display: flex; }
.barra .marca { width: 100%; }
.contenido { max-width: 70rem; margin: 0 auto; padding: 1rem 1rem 1.25rem; }
```

### 🧩 Archivo 7.11 · `index.html`

**Crear:** en `src` → nuevo archivo `index.html` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/index.html`

```html
<!doctype html>
<html lang="es">
<head>
  <meta charset="utf-8">
  <title>Tarjetas Bip!</title>
  <base href="/">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="icon" type="image/x-icon" href="favicon.ico">
</head>
<body>
  <app-root></app-root>
</body>
</html>
```


#### 🚀 Levantar y ✅ verificar (recorrido completo del CRUD)

1. Levanta el **backend** (`.\mvnw.cmd spring-boot:run` en `tarjetas-backend`).
2. Levanta el front: `npm start` → http://localhost:4200

| # | Acción | Resultado esperado |
|---|---|---|
| 1 | Entrar a la página | Redirige a `/tarjetas`: 3 cajas de resumen (3 tarjetas, 2 activas, saldo total $9.700) y las tarjetas rojas |
| 2 | Escribir "juan" en el buscador | Queda solo la tarjeta de Juan |
| 3 | Filtrar **Bloqueadas** | Solo la de Rosa (tarjeta gris) |
| 4 | **+ Nueva tarjeta** con número `123` y **Registrar** | Mensaje de validación en el campo; no se envía |
| 5 | Registrar número `12345678` | Aviso rojo *"Ya existe una tarjeta registrada con el número 12345678"* y el formulario **sigue abierto** |
| 6 | Registrar un número nuevo (ej. `99887766`) | Aviso verde, vuelve al listado y la tarjeta aparece primero |
| 7 | **Editar** una tarjeta | El número aparece bloqueado; cambias el saldo y guardas |
| 8 | **Eliminar** → Cancelar | Se cierra el diálogo, no pasa nada |
| 9 | **Eliminar** → Eliminar | La tarjeta desaparece y aparece un aviso |
| 10 | Apagar el backend y recargar | Mensaje *"No hay conexión con el servidor…"* con botón **Reintentar** |
| 11 | Comprobar en Swagger o `psql` | Los cambios están en la base de datos |


---

## Paso 8 · Tests

**🎓 Teoría.** Los tests usan `HttpTestingController`: **simulan** el backend, así se prueba sin servidor. El test del store verifica el resumen, los filtros, que POST/PUT/DELETE se llamen con el método correcto, que tras cada operación se recargue la lista, que un error 409 **no** emita y que un servidor apagado deje el mensaje de error. El test de `App` navega a `/tarjetas` y comprueba que se vean las tarjetas que entrega la API.

### 🧩 Archivo 8.1 · `tarjetas.store.spec.ts`

**Crear:** en `src/app/features/tarjetas` → nuevo archivo `tarjetas.store.spec.ts` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/app/features/tarjetas/tarjetas.store.spec.ts`

```ts
import { TestBed } from '@angular/core/testing';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';

import { errorInterceptor } from '../../core/interceptors/error.interceptor';
import { Tarjeta } from '../../core/models/tarjeta.model';
import { TarjetasStore } from './tarjetas.store';

const ahora = '2026-10-15T10:00:00';
const ana: Tarjeta = {
  id: 1, numero: '11111111', titular: 'Ana', alias: null, tipo: 'NORMAL', saldo: 1000,
  estado: 'ACTIVA', creadaEn: ahora, actualizadaEn: ahora,
};
const luis: Tarjeta = {
  id: 2, numero: '22222222', titular: 'Luis', alias: 'Uni', tipo: 'TNE_SUPERIOR', saldo: 500,
  estado: 'BLOQUEADA', creadaEn: ahora, actualizadaEn: ahora,
};

describe('TarjetasStore', () => {
  let store: TarjetasStore;
  let http: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({ providers: [provideHttpClient(withInterceptors([errorInterceptor])), provideHttpClientTesting()] });
    store = TestBed.inject(TarjetasStore);
    http = TestBed.inject(HttpTestingController);
  });

  afterEach(() => http.verify());

  function cargarDatos(): void {
    store.cargar();
    http.expectOne('/api/tarjetas').flush([ana, luis]);
  }

  it('carga las tarjetas', () => {
    store.cargar();
    expect(store.cargando()).toBe(true);

    http.expectOne('/api/tarjetas').flush([ana, luis]);

    expect(store.tarjetas().length).toBe(2);
    expect(store.cargando()).toBe(false);
  });

  it('calcula el resumen: total, activas y saldo', () => {
    cargarDatos();
    expect(store.resumen()).toEqual({ total: 2, activas: 1, saldoTotal: 1500 });
  });

  it('guarda el mensaje si el servidor no responde', () => {
    store.cargar();
    http.expectOne('/api/tarjetas').error(new ProgressEvent('error'), { status: 0 });

    expect(store.error()).toContain('No hay conexión');
    expect(store.cargando()).toBe(false);
  });

  it('filtra por estado y por texto', () => {
    cargarDatos();

    store.filtrarPorEstado('BLOQUEADA');
    expect(store.tarjetasFiltradas()).toEqual([luis]);

    store.filtrarPorEstado('TODAS');
    store.buscar('ana');
    expect(store.tarjetasFiltradas()).toEqual([ana]);
  });

  it('registra con POST y recarga la lista', () => {
    cargarDatos();
    let terminado = false;

    store
      .crear({ numero: '33333333', titular: 'Eva', alias: null, tipo: 'NORMAL', saldo: 100 })
      .subscribe(() => (terminado = true));

    const post = http.expectOne('/api/tarjetas');
    expect(post.request.method).toBe('POST');
    post.flush({ ...ana, id: 3, numero: '33333333', titular: 'Eva' });
    http.expectOne('/api/tarjetas').flush([ana, luis, { ...ana, id: 3 }]);

    expect(terminado).toBe(true);
    expect(store.tarjetas().length).toBe(3);
  });

  it('si el servidor rechaza (409) muestra el error y no emite', () => {
    cargarDatos();
    let terminado = false;

    store
      .crear({ numero: '11111111', titular: 'Ana', alias: null, tipo: 'NORMAL', saldo: 0 })
      .subscribe(() => (terminado = true));

    http.expectOne('/api/tarjetas').flush({ mensaje: 'Ya existe una tarjeta' }, { status: 409, statusText: 'Conflict' });

    expect(terminado).toBe(false);
  });

  it('edita con PUT y elimina con DELETE', () => {
    cargarDatos();

    store.actualizar(1, { titular: 'Ana D', alias: null, tipo: 'NORMAL', saldo: 5, estado: 'ACTIVA' }).subscribe();
    const put = http.expectOne('/api/tarjetas/1');
    expect(put.request.method).toBe('PUT');
    put.flush(ana);
    http.expectOne('/api/tarjetas').flush([ana, luis]);

    store.eliminar(2).subscribe();
    const del = http.expectOne('/api/tarjetas/2');
    expect(del.request.method).toBe('DELETE');
    del.flush(null);
    http.expectOne('/api/tarjetas').flush([ana]);

    expect(store.tarjetas()).toEqual([ana]);
  });
});
```

### 🧩 Archivo 8.2 · `app.spec.ts`

**Crear:** en `src/app` → nuevo archivo `app.spec.ts` *(VS Code: clic derecho en la carpeta → New File)*. **Pega este código completo:**

📄 `src/app/app.spec.ts`

```ts
import { TestBed } from '@angular/core/testing';
import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { provideRouter } from '@angular/router';
import { RouterTestingHarness } from '@angular/router/testing';

import { App } from './app';
import { routes } from './app.routes';

describe('App', () => {
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [App],
      providers: [provideRouter(routes), provideHttpClient(), provideHttpClientTesting()],
    }).compileComponents();
  });

  it('muestra la cabecera y las tarjetas que entrega la API', async () => {
    const http = TestBed.inject(HttpTestingController);
    const harness = await RouterTestingHarness.create();
    await harness.navigateByUrl('/tarjetas');

    http.expectOne('/api/tarjetas').flush([
      {
        id: 1,
        numero: '12345678',
        titular: 'María González',
        alias: 'Trabajo',
        tipo: 'NORMAL',
        saldo: 5200,
        estado: 'ACTIVA',
        creadaEn: '2026-10-15T10:00:00',
        actualizadaEn: '2026-10-15T10:00:00',
      },
    ]);
    harness.detectChanges();

    const texto = (harness.routeNativeElement as HTMLElement).textContent ?? '';
    expect(texto).toContain('Mis tarjetas Bip!');
    expect(texto).toContain('María González');
    expect(texto).toContain('1234 5678');
  });
});
```


```powershell
npx ng test --watch=false
```

**✅ Verificar:** `Test Files 2 passed` · `Tests 8 passed`.

Compilar para producción: `npx ng build` (genera `dist/tarjetas-frontend`).

## Checklist final

- [ ] Los componentes de `components/` y `shared/ui` no inyectan servicios de negocio: solo `input()` y `output()`.
- [ ] Solo las páginas conocen al store; solo el store conoce al api.
- [ ] Las rutas HTTP están únicamente en `core/api/tarjeta.api.ts`.
- [ ] El backend se alcanza por `proxy.conf.json` (sin CORS).
- [ ] `npx ng test` en verde y `npx ng build` sin errores.

## Problemas frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| *"No hay conexión con el servidor"* | Backend apagado o puerto distinto | Levantar el backend o cambiar el `target` de `proxy.conf.json` y **reiniciar** `npm start` |
| `http proxy error: ECONNREFUSED` en la terminal de Angular | Nada escucha en el puerto del proxy | Mismo caso anterior |
| Los cambios del proxy no se notan | El proxy se lee solo al arrancar | Detener y volver a ejecutar `npm start` |
| `NullInjectorError: No provider for HttpClient` | Falta `provideHttpClient` | Revisar `app.config.ts` (Paso 3) |
| La lista no se actualiza | Se mutó el array en vez de usar `signal.set` / `update` con uno nuevo | Crear siempre un array nuevo |
| El `id` llega `undefined` en la edición | Falta `withComponentInputBinding()` | Revisar `app.config.ts` |
| `Port 4200 is already in use` | Otro programa usa el puerto | `npx ng serve --port 4300` (no requiere tocar el backend: el proxy evita CORS) |

Volver al [README](../README.md)

