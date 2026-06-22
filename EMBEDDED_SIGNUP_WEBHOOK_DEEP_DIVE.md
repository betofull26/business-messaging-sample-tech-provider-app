# Documentación Técnica: Embedded Signup y Webhooks de WhatsApp Business

Este documento detalla la arquitectura y el flujo lógico para la integración de WhatsApp Business en una plataforma de Proveedor Tecnológico (Tech Provider).

## 1. Embedded Signup (Onboarding de Clientes)

El proceso de Embedded Signup permite que empresas externas vinculen sus cuentas de WhatsApp a tu aplicación de forma fluida.

### A. Frontend: Interfaz y Captura de Datos (`app/components/Fbl4bLauncher.tsx`)
El frontend es responsable de iniciar la interacción con Meta y capturar los metadatos de la sesión.

*   **SDK de Facebook**: Se utiliza el SDK de JavaScript oficial. La llamada a `FB.login` se configura con `response_type: 'code'`. Esto garantiza que el intercambio final del token ocurra en el servidor, manteniendo el `App Secret` a salvo.
*   **Escucha de Mensajes (`window.postMessage`)**: Mientras la ventana de registro de Meta está abierta, esta envía mensajes a la ventana madre. El componente implementa un listener para capturar el evento `WA_EMBEDDED_SIGNUP`.
    *   *Importancia*: Este evento contiene los IDs de la WABA, el número de teléfono y las páginas seleccionadas **antes** de que el flujo se complete formalmente.

### B. Backend: Intercambio y Registro (`app/api/token/route.ts`)
El backend recibe el `code` temporal y los metadatos para realizar las operaciones críticas.

1.  **Intercambio de Token (`getToken`)**: Utiliza el `code`, `client_id` y `client_secret` para obtener un *User Access Token* de larga duración.
2.  **Persistencia en Base de Datos**: Se almacenan los tokens vinculados a cada activo (WABA, Páginas, etc.). En este repositorio, se usa PostgreSQL con una lógica de `INSERT ... ON CONFLICT UPDATE`.
3.  **Configuración Automática**:
    *   **Registro (`registerNumber`)**: El servidor registra el número de teléfono en la infraestructura de WhatsApp usando el PIN de registro configurado.
    *   **Suscripción de Webhook (`subscribeWebhook`)**: Se suscribe la WABA a la aplicación para que los eventos (mensajes entrantes) empiecen a fluir hacia tu endpoint.

---

## 2. Lógica de Webhooks (Mensajería y Eventos)

El Webhook es el sistema de notificaciones "push" que recibe mensajes y cambios de estado.

### A. Verificación del Endpoint (GET)
Meta valida tu servidor enviando un reto (`challenge`). Solo si tu servidor conoce el `FB_VERIFY_TOKEN` secreto, puede responder correctamente.
*   **Archivo**: `app/api/webhooks/route.ts`

### B. Recepción y Seguridad (POST)
Es el componente más sensible a nivel de seguridad.

*   **Validación de Firma (`x-hub-signature-256`)**:
    *   Meta firma el cuerpo de cada petición usando tu `App Secret`.
    *   Tu servidor DEBE recalcular este HMAC SHA256 y comparar las firmas. Esto evita que atacantes inyecten mensajes falsos en tu sistema.
*   **Procesamiento del Payload**: El sistema procesa objetos del tipo `whatsapp_business_account`.
    *   **Mensajes**: Extrae el remitente (`from`) y el contenido (`text.body`).
    *   **Llamadas**: Maneja eventos de señalización para llamadas de voz/video.
    *   **Estados**: Puede procesar confirmaciones de lectura, entrega y errores.

---

## 3. Resumen de Archivos Críticos

| Componente | Ruta del Archivo | Función Principal |
| :--- | :--- | :--- |
| **Frontend Launcher** | `app/components/Fbl4bLauncher.tsx` | Inicializa SDK y lanza el pop-up de Meta. |
| **Backend Exchange** | `app/api/token/route.ts` | Intercambia el código por un token y registra el número. |
| **Webhook Handler** | `app/api/webhooks/route.ts` | Valida firmas y procesa mensajes entrantes. |
| **Graph API Utils** | `app/api/beUtils.ts` | Contiene los wrappers para llamadas a la Graph API. |

---

## 4. Guía de Adaptación para tu Sistema

1.  **Seguridad**: Copia estrictamente la lógica de validación de firma HMAC SHA256 de `app/api/webhooks/route.ts`. No aceptes webhooks sin validar.
2.  **Tokens**: Recuerda que los tokens de usuario deben ser almacenados de forma segura. Estos tokens te permiten enviar mensajes "en nombre de" tu cliente.
3.  **Configuración en Meta**: Asegúrate de que en el Dashboard de Meta, la URL de tu Webhook apunte a tu endpoint POST y que el campo `messages` esté marcado en la configuración de WhatsApp.
