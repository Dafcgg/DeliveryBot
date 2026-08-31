# Proyecto DeliveryBot — Dilan Fonseca

Sistema de gestión de pedidos internos para cafeterías, construido sobre **n8n**, **Telegram** y **Google Sheets**, con un agente de Inteligencia Artificial conversacional (Google Gemini) como núcleo de interacción.

---

## 📋 Descripción del proyecto

DeliveryBot es una terminal de pedidos inteligente pensada para oficinas, universidades o centros de trabajo. A través de un bot de Telegram con **botones interactivos (inline keyboard)**, los usuarios pueden explorar el menú de la cafetería por categorías, elegir productos con cantidad exacta, armar un carrito, confirmar su pedido, seleccionar método de pago y recibir un número de orden — mientras el personal de cocina recibe cada pedido confirmado en tiempo real, en un grupo de Telegram dedicado.

Toda la información operativa (menú, pedidos y sesiones activas) se centraliza en una única base de datos en Google Sheets, sin necesidad de una app dedicada ni infraestructura de backend propia.

##  Objetivos

- Digitalizar y agilizar el proceso de pedidos internos de una cafetería.
- Reducir errores humanos en la toma de pedidos y el control de stock en tiempo real.
- Centralizar la información operativa en una única fuente de datos (Google Sheets), accesible y editable sin conocimientos técnicos.
- Ofrecer una experiencia conversacional natural y guiada mediante un agente de IA con botones interactivos, sin necesidad de una app dedicada.
- Notificar en tiempo real al personal de cocina sobre nuevos pedidos confirmados, con el mismo detalle que recibe el cliente.
- Registrar el método de pago elegido por el usuario como parte del historial de cada pedido.

---

## 🏗️ Arquitectura del sistema

El flujo principal (`DeliveryBot - Flujo principal.json`) se construye en n8n con los siguientes módulos:

| Módulo | Nodo n8n | Función |
|---|---|---|
| Entrada | Telegram Trigger | Captura cada mensaje entrante del usuario: texto libre o clics en botones (callback_query). |
| Normalización | Set ("Normalizar Entrada") | Extrae de forma unificada `chat_id`, `texto_usuario` y `nombre_usuario`, sin importar si el evento vino de un mensaje de texto o de un botón presionado. |
| Cerebro conversacional | AI Agent (Tools Agent) | Interpreta la intención del usuario y ejecuta el flujo wizard completo: Bienvenida → Explorar Menú → Selección de producto → Cantidad → Confirmación de Carrito → Método de Pago → Procesamiento. |
| Modelo de lenguaje | Chat Model (Google Gemini) | Motor de lenguaje que impulsa al AI Agent. |
| Memoria | Memory (Buffer Window) | Mantiene el contexto de la conversación durante la sesión activa del usuario, indexado por `chat_id`. |
| Herramientas | 5 Tools (Google Sheets) | Leer Menu, Gestionar Sesion, Descontar Stock, Registrar Pedido, Consultar Estado Pedido. |
| Parseo de salida | Code ("Parsear Respuesta IA") | Convierte la respuesta JSON estructurada del agente (`mensaje` + `botones`) en un objeto listo para construir el teclado interactivo de Telegram, con manejo de errores si el modelo no respeta el formato. |
| Salida al usuario | HTTP Request ("Telegram - Responder Usuario") | Envía el mensaje y el `inline_keyboard` dinámico al usuario vía la API de Telegram. |
| Salida a cocina | HTTP Request ("Telegram - Notificar Cocina") | Cuando el mensaje generado corresponde a un pedido confirmado (contiene el ID `ORD-2026-...`), reenvía exactamente el mismo mensaje a un grupo de Telegram dedicado a cocina. |

### Por qué HTTP Request en lugar del nodo Telegram nativo

El nodo Telegram estándar de n8n solo permite definir un `inline_keyboard` fijo, configurado a mano en la interfaz — no puede recibir una cantidad variable de botones generados dinámicamente por el modelo de IA (por ejemplo, la cantidad de productos de una categoría, que cambia según el inventario). Por eso los nodos de salida usan **HTTP Request** apuntando directamente al endpoint `sendMessage` de la API de Telegram, con el `reply_markup` construido en tiempo real por el nodo Code a partir de la respuesta del agente.

---

## 🗄️ Base de datos (Google Sheets)

**Nombre del archivo:** `Menu_DeliveryBot`
**Enlace:** _[(https://docs.google.com/spreadsheets/d/1-j2ZwE1iFPiv1cp8hjZiiVMHKUjBHKeSzMLaAhIKBxc/edit?usp=sharing)]_

### Pestaña `MENU`
| Columna | Tipo | Descripción |
|---|---|---|
| `id_item` | Texto | Identificador único del producto (ej. `M001`). Usado como clave en todas las operaciones. |
| `categoria` | Texto | Categoría del producto (ej. Café, Bebidas frías, Snacks, Panadería). |
| `nombre` | Texto | Nombre visible del producto. |
| `descripcion` | Texto | Descripción breve. |
| `precio` | Número | Precio unitario, sin símbolo de moneda. |
| `stock` | Número | Unidades disponibles. Se valida antes de cada venta y se descuenta al confirmar un pedido. |
| `activo` | Booleano | Indica si el producto debe mostrarse en el menú. |

### Pestaña `PEDIDOS`
| Columna | Tipo | Descripción |
|---|---|---|
| `id_pedido` | Texto | Formato `ORD-2026-XXXXXX`, generado por el agente al confirmar. |
| `id_usuario` | Texto | `chat_id` de Telegram del cliente. |
| `fecha_hora` | Texto | Fecha y hora del registro. |
| `items` | Texto (JSON) | Detalle del carrito: producto, cantidad y precio unitario de cada línea. |
| `total` | Número | Total a pagar del pedido. |
| `estado` | Texto | Estado del pedido (`recibido`, `en preparación`, `listo`, `entregado`). |
| `notas` | Texto | Método de pago elegido por el cliente (efectivo, tarjeta o transferencia). |

### Pestaña `SESSIONS`
| Columna | Tipo | Descripción |
|---|---|---|
| `chat_id` | Texto | Identificador único de la sesión activa del usuario. |
| `estado_conversacion` | Texto | Paso actual del wizard (`explorando`, `esperando_cantidad`, `inicio`, etc.), usado para que el agente sepa cómo interpretar el siguiente mensaje del usuario. |
| `carrito_temp` | Texto (JSON) | Carrito de compras mientras el pedido no ha sido confirmado. |
| `ultimo_item_seleccionado` | Texto | `id_item` del producto que el usuario seleccionó y sobre el cual se está preguntando la cantidad. |
| `timestamp_actualizacion` | Texto | Marca de tiempo de la última modificación de la sesión. |

---

## 🔄 Flujo conversacional completo

1. **Bienvenida (`/start`)**: el bot saluda y presenta las categorías disponibles como botones con emoji (ej. `☕ Bebidas`, `🍽️ Comida`).
2. **Comando `/menu`**: reabre el listado de categorías en cualquier momento de la conversación.
3. **Explorar categoría**: al elegir una categoría, se listan sus productos (nombre, descripción, precio) cada uno con su emoji específico y su propio botón, más un botón para volver a categorías.
4. **Selección de producto**: al tocar un producto, el bot pregunta la cantidad deseada, ofreciendo botones rápidos (1 a 4) y aceptando también que el usuario escriba cualquier número directamente en el chat. Incluye un botón "Volver" para regresar a la lista de productos sin agregar nada.
5. **Validación y carrito**: se valida el stock disponible antes de agregar el producto con la cantidad elegida; si no alcanza, se informa el máximo disponible.
6. **Repetir o finalizar**: tras cada producto agregado, se pregunta si desea agregar otro o finalizar el pedido.
7. **Confirmación del carrito**: se muestra el resumen completo (cantidad × producto, subtotales y total) con botones "Confirmar" y "Cancelar".
8. **Método de pago**: al confirmar, se pregunta el método de pago (efectivo, tarjeta o transferencia) antes de procesar nada.
9. **Procesamiento del pedido**: al elegir el método de pago, se genera el `id_pedido`, se descuenta el stock, se registra el pedido en `PEDIDOS` (con el método de pago en la columna `notas`) y se limpia el carrito temporal.
10. **Notificación a cocina**: el mismo mensaje de confirmación que recibe el cliente se reenvía automáticamente al grupo de Telegram de cocina.
11. **Comando `/estado`**: consulta el pedido más reciente del usuario y devuelve su estado actual.
12. **Comando `/cancelar`**: vacía el carrito temporal activo, si existe.
13. **Comando `/ayuda`**: explica brevemente las funciones disponibles del bot.

---

## ⚙️ Guía de instalación y configuración

### Requisitos previos
- Cuenta de n8n (cloud o self-hosted).
- Bot de Telegram creado vía [@BotFather](https://t.me/BotFather) (token del bot).
- Un grupo de Telegram destinado a "cocina", con el bot agregado como miembro, para recibir las notificaciones.
- Cuenta de Google con acceso a Google Sheets y credenciales OAuth2 configuradas en n8n.
- Cuenta y API Key de Google Gemini para el Chat Model del agente.

### Pasos

1. **Base de datos**: crea o duplica la hoja `Menu_DeliveryBot` con las pestañas `MENU`, `PEDIDOS` y `SESSIONS`, con las columnas exactas descritas arriba, y cárgala con datos de prueba.
2. **Credenciales en n8n**: configura la credencial de Telegram API (token del bot), la credencial de Google Sheets OAuth2, y la credencial de Google Gemini.
3. **Importar el workflow**: importa `DeliveryBot - Flujo principal.json` en tu instancia de n8n.
4. **Conectar cada nodo de Google Sheets Tool** (Leer Menu, Gestionar Sesion, Descontar Stock, Registrar Pedido, Consultar Estado Pedido) a la salida "Tool" del nodo AI Agent, y selecciona el Document y el Sheet de cada uno desde la lista ("From list"), en vez de dejarlos por nombre, para evitar errores de referencia al importar.
5. **Configurar los nodos de salida a Telegram** (`Telegram - Responder Usuario` y `Telegram - Notificar Cocina`, ambos de tipo HTTP Request): define la URL completa `https://api.telegram.org/bot<TU_TOKEN>/sendMessage` y reemplaza el chat_id de cocina por el ID real de tu grupo (obtenido vía `getUpdates` de la API de Telegram).
6. **Pegar el System Prompt** definitivo en el campo "System Message" del nodo AI Agent (ver sección siguiente).
7. **Activar el workflow** y probar enviando `/start` al bot.

### Obtener el Chat ID del grupo de cocina
1. Agrega el bot al grupo de Telegram de cocina.
2. Envía cualquier mensaje al grupo.
3. Visita `https://api.telegram.org/bot<TU_TOKEN>/getUpdates` en el navegador.
4. Busca el campo `chat.id` correspondiente al grupo (número negativo).

---

## 🧠 System Prompt del AI Agent

El texto completo del prompt del sistema —con las reglas de formato de salida (JSON con `mensaje` y `botones`), el flujo conversacional detallado paso a paso, las reglas de uso de herramientas y las restricciones de negocio— se encuentra documentado en el archivo `01_system_prompt.md` incluido en este repositorio, y debe copiarse tal cual en el campo "System Message" del nodo AI Agent en n8n.

---

## 🧪 Pruebas realizadas

- Flujo completo: `/start` → elegir categoría → elegir producto → definir cantidad (por botón y por texto libre) → confirmar carrito → elegir método de pago → recibir ID de pedido.
- Validación de stock insuficiente al seleccionar una cantidad mayor a la disponible.
- Botón "Volver" desde la pantalla de cantidad y desde la lista de productos.
- Notificación a cocina con el mismo contenido exacto que recibe el cliente.
- Comando `/estado` con y sin pedidos previos.
- Comando `/cancelar` con carrito activo y sin carrito activo.
- Comando `/ayuda`.

---
