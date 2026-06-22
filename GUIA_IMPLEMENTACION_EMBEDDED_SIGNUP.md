# Guía de Implementación: WhatsApp Embedded Signup

Esta guía detalla el flujo exacto para implementar el **Embedded Signup** de WhatsApp en tu propia aplicación, basado en el código de referencia de este repositorio.

---

## 1. Requisitos en Meta Developer Dashboard

Antes de tocar el código, debes configurar tu aplicación en [developers.facebook.com](https://developers.facebook.com):

1.  **Crear una App**: Elige el caso de uso "Connect through WhatsApp".
2.  **Configurar Facebook Login for Business**:
    *   Añade tus dominios en "Allowed Domains for the JavaScript SDK".
    *   Añade tu URL de redirección en "Valid OAuth Redirect URIs".
3.  **Configuración de Tech Provider**: Crea un `config_id` en la sección de WhatsApp. Este ID identifica tu configuración de permisos ante Meta.

---

## 2. Base de Datos (Esquema SQL)

Necesitas persistir los tokens de tus clientes. Usa este esquema básico (PostgreSQL):

```sql
CREATE TABLE wabas (
  waba_id BIGINT PRIMARY KEY,
  user_id VARCHAR NOT NULL, -- ID de tu usuario en tu app
  access_token TEXT NOT NULL,
  business_id BIGINT,
  last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE phones (
  phone_id BIGINT PRIMARY KEY,
  waba_id BIGINT REFERENCES wabas(waba_id),
  display_phone_number VARCHAR
);
```

---

## 3. Implementación en el Frontend (React/Next.js)

### Paso A: Cargar el SDK e Inicializar
Debes cargar el SDK de Facebook y ejecutar `FB.init`.

```javascript
// app/components/Fbl4bLauncher.tsx
FB.init({
  appId: 'TU_FB_APP_ID',
  version: 'v24.0', // Usa la versión más reciente
});
```

### Paso B: Escuchar los eventos de la ventana
Meta envía los IDs de la WABA y el teléfono a través de eventos de mensaje mientras el pop-up está abierto.

```javascript
window.addEventListener('message', (event) => {
  if (!event.origin.endsWith('facebook.com')) return;
  const data = JSON.parse(event.data);

  if (data.type === 'WA_EMBEDDED_SIGNUP') {
    // Estos datos contienen los IDs seleccionados por el usuario
    console.log("IDs recibidos:", data.data);
    this.sessionInfo = data;
  }
});
```

### Paso C: Lanzar el Login
Usa `response_type: 'code'` para obtener un código que el backend intercambiará por el token.

```javascript
const launchSignup = () => {
  const config = {
    config_id: 'TU_CONFIG_ID',
    response_type: 'code',
    override_default_response_type: true,
    extras: { sessionInfoVersion: '3' }
  };

  FB.login((response) => {
    if (response.authResponse) {
      const code = response.authResponse.code;
      // Envía 'code' y 'sessionInfo' a tu API de backend
      enviarAlBackend(code, this.sessionInfo);
    }
  }, config);
};
```

---

## 4. Implementación en el Backend (Node.js)

### Paso A: Intercambiar el Código por el Token
Usa el `App Secret` para validar el código y obtener el token de larga duración.

```javascript
// app/api/beUtils.ts
async function getToken(code, appId, appSecret) {
  const url = `https://graph.facebook.com/v22.0/oauth/access_token?client_id=${appId}&client_secret=${appSecret}&code=${code}`;
  const res = await fetch(url);
  const data = await res.json();
  return data.access_token;
}
```

### Paso B: Registro y Suscripción
Una vez obtenido el token, debes activar la cuenta:

1.  **Guardar en DB**: Guarda el token asociado al `waba_id`.
2.  **Suscribir Webhook**: Notifica a Meta que tu app gestionará esta WABA.
    ```javascript
    await fetch(`https://graph.facebook.com/v22.0/${wabaId}/subscribed_apps`, {
      method: 'POST',
      headers: { 'Authorization': `Bearer ${token}` }
    });
  ```
3.  **Registrar Teléfono**: Si el usuario seleccionó un número nuevo, regístralo.

---

## 5. Resumen del Flujo de Datos

1.  **App** → Lanza pop-up de Meta.
2.  **Usuario** → Elige su cuenta y da permisos.
3.  **Meta** → Envía IDs a la App (vía `postMessage`) y devuelve `code` (vía callback).
4.  **App** → Envía `code` + `IDs` al **Backend**.
5.  **Backend** → Cambia `code` por `Access Token` (usando `App Secret`).
6.  **Backend** → Guarda en **DB** y activa Webhooks.

¡Listo! Con este flujo, tu app ahora tiene permiso para enviar mensajes y recibir eventos de ese cliente de forma segura.
