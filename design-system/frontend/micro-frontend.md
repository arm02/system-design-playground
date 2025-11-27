# System Design: Micro-Frontend Architecture

## 1. Overview

Micro-frontend adalah pendekatan arsitektur di mana aplikasi frontend
besar dipecah menjadi beberapa bagian kecil yang dapat dikembangkan,
diuji, dan dideploy secara independen. Konsep ini terinspirasi dari
microservices pada backend.

### Tujuan

-   Skalabilitas tim dan aplikasi
-   Pengembangan paralel
-   Deploy independen
-   Isolasi teknologi

------------------------------------------------------------------------

## 2. Arsitektur Umum Micro-Frontend

### Penjelasan

Arsitektur micro-frontend terdiri dari beberapa aplikasi kecil
(micro-app) yang digabungkan menjadi satu aplikasi besar melalui
container/shell. Integrasi bisa dilakukan dengan berbagai teknik seperti
Module Federation, Web Components, iframe, atau build-time integration.

### Diagram (Markdown)

``` mermaid
flowchart LR
    A[Shell / Container] --> B[Micro App 1]
    A --> C[Micro App 2]
    A --> D[Micro App 3]
```

------------------------------------------------------------------------

## 3. Pendekatan Integrasi Micro-Frontend

### 3.1 Module Federation (Webpack 5)

**Penjelasan:**\
Module Federation memungkinkan aplikasi saling mengkonsumsi modul satu
sama lain secara runtime tanpa perlu build bersama.

**Contoh konfigurasi:**

``` js
// webpack.config.js
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: "app1",
      exposes: {
        "./Button": "./src/Button",
      },
      remotes: {
        app2: "app2@http://localhost:3002/remoteEntry.js"
      }
    })
  ]
}
```
Module Federation adalah teknik runtime module sharing yang memungkinkan
aplikasi berbagi komponen, hooks, services, atau utilitas antar aplikasi
tanpa rebuild seluruh sistem.

**Cara Kerja Module Federation**
1.  **Host (Shell)** memuat remote container dari aplikasi lain.
2.  **Remote** mengekspos modul melalui `ModuleFederationPlugin`.
3.  Ketika host meminta modul remote, Webpack mengambil file
    `remoteEntry.js`.
4.  RemoteEntry berperan sebagai "runtime manifest" yang berisi daftar
    modul yang bisa di-load.
5.  Modul dikonsumsi secara **dynamic runtime import** tanpa rebuild.


**Struktur Module Federation**
**Host (Shell)**
``` js
new ModuleFederationPlugin({
  remotes: {
    cart: "cart@http://localhost:3002/remoteEntry.js",
  }
})
```

**Remote (Cart App)**
``` js
new ModuleFederationPlugin({
  name: "cart",
  filename: "remoteEntry.js",
  exposes: {
    "./CartWidget": "./src/components/CartWidget",
  }
})
```

**Load Dynamic Module**

``` js
import("cart/CartWidget").then((module) => {
  const CartWidget = module.default;
});
```

**Sharing Dependencies**
Untuk mencegah **duplicate React**, gunakan shared singleton:
``` js
shared: {
  react: { singleton: true, requiredVersion: "^18.0.0" },
  "react-dom": { singleton: true },
}
```
**Manfaat Module Federation**

  |Kategori|Detail|
  |--------|------|
  |Performance|Load on demand & shared libraries|
  |Flexibility|Bisa mix framework jika diperlukan|
  |DX|Aplikasi bisa dideploy mandiri|
  |Scalability|Tim dapat bekerja paralel|

**Tantangan Module Federation**

  |Tantangan|Penjelasan|
  |---------|----------|
  |Tight coupling|Jika terlalu banyak shared dependency|
  |Version mismatch|React versi berbeda dapat menyebabkan error|
  |Observability|Debugging runtime module cukup kompleks|
  |Latency|RemoteEntry harus stabil dan cepat diload|

**Pola Arsitektur Module Federation**
1. **Host-Single Remote**
Sederhana, 1 host -- 1 remote.

2. **Host-Multiple Remote (paling umum)**
Host mengoordinasi banyak micro-app.

3. **Micro-App sebagai Host & Remote**
Aplikasi saling mengonsumsi modul satu sama lain.

``` mermaid
flowchart LR
    A[App A - Host & Remote]
    B[App B - Host & Remote]
    A <-->|shared modules| B
```
---------------------

### 3.2 Web Components

-   Standar browser
-   Framework-agnostic\
    Contoh:

``` js
class MyWidget extends HTMLElement {
  connectedCallback() {
    this.innerHTML = "<h1>Hello</h1>";
  }
}
customElements.define("my-widget", MyWidget);
```

### 3.3 Iframe

Cocok untuk isolasi total tetapi overhead tinggi.

#### Tabel Perbandingan Integrasi

  | Integrasi | Kelebihan | Kekurangan |
  |-----------|-----------|------------|
  | Module Federation|Sharing code, kinerja cepat|Kompleks setup|
  | Web Components|Agnostic framework|Limited state sharing|
  | Iframe|Isolasi kuat|Berat & kurang seamless|

---------------------------

## 4. Komunikasi Antar Micro-Frontend

### Teknik Komunikasi

1.  **Custom Events**
2.  **Shared global event bus**
3.  **URL-based communication**
4.  **State sync via Module Federation**

### Contoh Custom Event

``` js
// dispatch
window.dispatchEvent(new CustomEvent("cart:update", { detail: { count: 3 } }));

// listen
window.addEventListener("cart:update", (e) => console.log(e.detail));
```

------------------------------------------------------------------------

## 5. Deployment Strategy

### Model Deployment

-   **Single Repo (Monorepo)**\
    Cocok untuk sinkronisasi antar tim.
-   **Multi Repo (Polyrepo)**\
    Cocok untuk kemandirian tim.

### Diagram Deployment

``` mermaid
flowchart TD
    A[CI/CD Micro App 1] --> Z[Production]
    B[CI/CD Micro App 2] --> Z
    C[CI/CD Micro App 3] --> Z
```

------------------------------------------------------------------------

## 6. Best Practices

-   Gunakan design system bersama
-   Tentukan kontrak API yang jelas antar micro-app
-   Hindari tight coupling
-   Monitoring dan observability sangat penting
-   Sediakan fallback jika salah satu micro-app gagal dimuat

------------------------------------------------------------------------

## 7. Kesimpulan

Micro-frontend adalah solusi efektif untuk aplikasi berskala besar yang
membutuhkan modularitas dan skalabilitas tim. Namun, implementasinya
harus mempertimbangkan kompleksitas orchestration dan communication.
