# Elsa Copilot - Guía de Funcionamiento del Proyecto

## 📋 Tabla de Contenidos

1. [¿Qué es Elsa Copilot?](#qué-es-elsa-copilot)
2. [Arquitectura General](#arquitectura-general)
3. [Componentes Principales](#componentes-principales)
4. [Flujo de Trabajo](#flujo-de-trabajo)
5. [Módulos del Proyecto](#módulos-del-proyecto)
6. [Configuración y Inicio](#configuración-y-inicio)
7. [API y Endpoints](#api-y-endpoints)
8. [Tecnologías Utilizadas](#tecnologías-utilizadas)
9. [Fases de Desarrollo](#fases-de-desarrollo)

---

## 🎯 ¿Qué es Elsa Copilot?

**Elsa Copilot** es una integración de asistencia por IA dentro de **Elsa Studio** y **Elsa Server** que permite:

- ✅ **Chat inteligente** con asistencia basada en GitHub Copilot SDK
- ✅ **Análisis de flujos de trabajo** (workflows) en tiempo real
- ✅ **Proposición de workflows** nuevos o modificados para revisión
- ✅ **Diagnóstico de errores** en instancias de workflow fallidas
- ✅ **Acceso a actividades** disponibles y su documentación

**Nota importante:** Toda la funcionalidad es **modular y opcional**. El proyecto funciona sin estas características de IA si no se necesitan.

---

## 🏗️ Arquitectura General

El proyecto utiliza una arquitectura **híbrida ASP.NET Core** que combina:

┌─────────────────────────────────────┐
│     Elsa Copilot Workbench          │
│  (ASP.NET Core + Blazor Server)     │
├─────────────────────────────────────┤
│                                     │
│  ┌──────────────┐  ┌──────────────┐│
│  │ Elsa Server  │  │ Elsa Studio  ││
│  │ (Backend)    │  │ (UI Blazor)  ││
│  │              │  │              ││
│  │ • Workflows  │  │ • Chat UI    ││
│  │ • APIs       │  │ • Componentes││
│  │ • Módulos    │  │ • Pages      ││
│  └──────────────┘  └──────────────┘│
│                                     │
│  ┌──────────────────────────────────┤
│  │   Módulos Personalizados         │
│  │   (Chat, Propuestas, etc.)       │
│  └──────────────────────────────────┘
└─────────────────────────────────────┘

### Capas del Proyecto:

1. **Elsa Server** → Motor de flujos de trabajo + APIs REST
2. **Elsa Studio** → Interfaz web (Blazor) para crear/editar workflows
3. **Módulos Custom** → Extensiones opcionales (Chat, Copilot, etc.)
4. **Base de Datos** → SQLite (local) o SQL Server (producción)

---

## 🔧 Componentes Principales

### 1. **Program.cs** (Punto de Entrada)

El archivo `src/Elsa.Copilot.Workbench/Program.cs` es el corazón de la aplicación:

```csharp
var builder = WebApplication.CreateBuilder(args);

// 1️⃣ Configurar Elsa Server (motor de workflows)
ElsaServerSetup.AddElsaServer(builder.Services, builder.Configuration);

// 2️⃣ Configurar Elsa Studio (interfaz Blazor)
ElsaStudioSetup.AddElsaStudio(builder.Services, builder.Configuration);

// 3️⃣ Registrar módulos personalizados (Chat, etc.)
ModuleRegistration.RegisterModules(builder.Services);
```

**¿Qué sucede aquí?**
- Se configura el contenedor de **inyección de dependencias** (DI)
- Se registran todos los servicios necesarios
- Se configuran los middlewares (autenticación, CORS, etc.)
- Se mapean los endpoints de la API

### 2. **Elsa Server Setup**

Configura el motor de workflows:
- Actividades disponibles (acciones que pueden realizar los workflows)
- Persistencia de workflows (almacenarlos en BD)
- APIs REST para crear/ejecutar workflows
- Autenticación y autorización

### 3. **Elsa Studio Setup**

Configura la interfaz visual:
- Componentes Blazor (interfaz interactiva)
- Páginas para diseñar workflows gráficamente
- Conexión con Elsa Server via HTTP

### 4. **Module Registration**

Registra módulos opcionales:
- **Chat Module** → Endpoints para chat con IA
- Futuros módulos de análisis, propuestas, etc.

---

## 🔄 Flujo de Trabajo (Paso a Paso)

### Escenario: Un usuario crea un workflow con ayuda de IA

1. Usuario abre Elsa Studio en el navegador
   ↓
2. Selecciona un workflow existente o crea uno nuevo
   ↓
3. Abre el chat lateral (componente Blazor)
   ↓
4. Escribe una pregunta: "¿Cómo envío un email en este workflow?"
   ↓
5. Studio envía: POST /copilot/chat
   Cuerpo:
   {
     "message": "¿Cómo envío un email?",
     "workflowDefinitionId": "my-workflow-123",
     "context": {...}
   }
   ↓
6. Elsa Server recibe la solicitud
   ↓
7. El Chat Module procesa:
   a) Valida autenticación (API Key o JWT)
   b) Carga el workflow del contexto
   c) Prepara las "herramientas" (tools):
      - GetWorkflowDefinition
      - GetActivityCatalog
      - GetWorkflowInstanceState
      - GetWorkflowInstanceErrors
   ↓
8. Envía a GitHub Copilot SDK con:
   - Pregunta del usuario
   - Contexto del workflow
   - Herramientas disponibles
   ↓
9. Copilot SDK procesa y responde
   ↓
10. Elsa Server transmite respuesta via Server-Sent Events (SSE)
    Formato: data: "Puedes usar la actividad SendEmail..."\n\n
    ↓
11. Studio recibe streaming en tiempo real
    ↓
12. Chat muestra la respuesta palabra por palabra
    ↓
13. Si Copilot propone cambios:
    - Studio muestra "Revisar propuesta"
    - Usuario aprueba o rechaza
    - Si aprueba → guarda el workflow via API estándar

---

## 📦 Módulos del Proyecto

### Estructura de Carpetas

```
src/
├── Elsa.Copilot.Workbench/           # Aplicación principal
│   ├── Program.cs                    # Punto de entrada
│   ├── Setup/                        # Configuración
│   ├── Pages/                        # Páginas Razor
│   ├── Components/                   # Componentes Blazor
│   └── appsettings.json              # Configuración
│
└── Modules/
    ├── Core/                         # Módulos de servidor
    │   └── Elsa.Copilot.Modules.Core.Chat/
    │       ├── Controllers/          # APIs REST
    │       ├── Services/             # Lógica de negocio
    │       ├── Tools/                # Herramientas para IA
    │       ├── Models/               # DTOs y modelos
    │       └── Extensions/           # Registro en DI
    │
    └── Studio/                       # Módulos de UI
        └── Elsa.Copilot.Modules.Studio.Placeholder/
            └── Componentes Blazor
```

### Módulo Chat (Core)

**Ubicación:** `src/Modules/Core/Elsa.Copilot.Modules.Core.Chat/`

**Responsabilidades:**
- Exponer endpoint REST `/copilot/chat`
- Procesar mensajes de chat
- Integrar GitHub Copilot SDK
- Ejecutar "herramientas" de IA

**Componentes:**

| Componente | Función |
|-----------|---------|
| `CopilotChatController` | Recibe solicitudes HTTP, maneja streaming SSE |
| `CopilotChatService` | Orquesta la comunicación con Copilot SDK |
| `MockChatClient` | Cliente de prueba que simula respuestas |
| `*Tool.cs` | Funciones que Copilot puede invocar para obtener datos |

---

## ⚙️ Configuración y Inicio

### 1. **Prerrequisitos**

- .NET 8 SDK
- Visual Studio 2022
- Git

### 2. **Clonar el Repositorio**

```bash
git clone https://github.com/elsa-workflows/elsa-copilot
cd elsa-copilot
```

### 3. **Restaurar Dependencias**

```bash
dotnet restore
```

### 4. **Construir el Proyecto**

```bash
dotnet build
```

### 5. **Ejecutar la Aplicación**

```bash
dotnet run --project src/Elsa.Copilot.Workbench/Elsa.Copilot.Workbench.csproj
```

**Salida esperada:**
```
info: Elsa.Server[0]
      Elsa Server started on https://localhost:7224
info: Microsoft.Hosting.Lifetime[14]
      Application started. Press Ctrl+C to exit.
```

### 6. **Acceder a la Aplicación**

- **Elsa Studio (UI):** `https://localhost:7224`
- **APIs:** `https://localhost:7224/api/...`

### 7. **Configuración**

Para detalles sobre variables de entorno, claves de seguridad y configuración por ambiente, consulta:
📄 [`src/Elsa.Copilot.Workbench/CONFIGURATION.md`](src/Elsa.Copilot.Workbench/CONFIGURATION.md)

**Configuración rápida (desarrollo):**
```bash
# Variable de entorno necesaria (mínimo 256 bits)
export Elsa__Identity__SigningKey="your-development-key-here"

# O ejecutar directamente
dotnet run --project src/Elsa.Copilot.Workbench/Elsa.Copilot.Workbench.csproj
```

---

## 🔌 API y Endpoints
    
### Endpoint Principal: Chat

**Ruta:** `POST /copilot/chat`

**Headers requeridos:**
Content-Type: application/json
Authorization: ApiKey [tu-api-key-aquí]

**Cuerpo (Request):**
```json
{
  "message": "¿Qué actividades necesito para validar un email?",
  "workflowDefinitionId": "send-email-workflow",
  "workflowInstanceId": null,
  "selectedActivityId": null
}
```

**Respuesta (Server-Sent Events):**
```
data: Para validar un email puedes usar:\n\n

data: 1. La actividad 'RegexMatch' para validar el formato\n

data: 2. La actividad 'HttpRequest' para verificar si existe\n

data: [DONE]
```

### Herramientas Disponibles (Tools)

La IA puede usar estas funciones automáticamente:

| Herramienta | Descripción |
|-------------|-------------|
| `GetWorkflowDefinition` | Obtiene la estructura completa de un workflow |
| `GetActivityCatalog` | Lista todas las actividades disponibles |
| `GetWorkflowInstanceState` | Inspecciona el estado de una instancia en ejecución |
| `GetWorkflowInstanceErrors` | Obtiene detalles de errores en instancias fallidas |

---

## 🛠️ Tecnologías Utilizadas

| Tecnología | Versión | Propósito |
|-----------|---------|----------|
| **.NET** | 8 | Framework principal |
| **ASP.NET Core** | 8 | Servidor web y APIs |
| **Blazor Server** | 8 | Interfaz de usuario interactiva |
| **Elsa Workflows** | 3.x | Motor de workflows |
| **Elsa Studio** | 3.x | Diseñador visual de workflows |
| **GitHub Copilot SDK** | Latest | Integración de IA |
| **Microsoft.Extensions.AI** | 9.0.1-preview.1 | Cliente de IA genérico |
| **SQLite** | (Embedded) | Base de datos por defecto |
| **Entity Framework Core** | 8 | Acceso a datos |

---

## 📅 Fases de Desarrollo

### ✅ **Fase 0: Fundación** (Completada)

- Crear solución ASP.NET Core híbrida
- Configurar Elsa Server + Elsa Studio
- Establecer estructura modular
- Documentar arquitectura

### ✅ **Fase 1: Chat de Solo Lectura** (En Progreso)

- ✅ Endpoint `/copilot/chat` operacional
- ✅ Streaming Server-Sent Events
- ✅ Herramientas de lectura implementadas
- ✅ Mock Chat Client para pruebas
- ✅ Autenticación integrada

**Características actuales:**
- El usuario puede hacer preguntas sobre workflows
- IA responde usando contexto del workflow
- Las respuestas se transmiten en tiempo real

### ⏳ **Fase 2: Propuestas de Workflows Nuevos** (Pendiente)

- Endpoint para crear workflows nuevos mediante IA
- Revisión de propuestas antes de aplicar
- Guardado en base de datos

### ⏳ **Fase 3: Propuestas de Ediciones** (Pendiente)

- Endpoint para modificar workflows existentes
- IA sugiere cambios en YAML/JSON
- Usuario revisa y aprueba cambios

### ⏳ **Fase 4: Diagnósticos en Runtime** (Pendiente)

- Explicación de errores en instancias fallidas
- Sugerencias de correcciones
- Análisis de rendimiento

---

## 🔐 Seguridad y Autorización

### Autenticación

- Usa el sistema de autenticación de **Elsa** existente
- Soporta **API Keys** y **JWT tokens**
- Configuración en `appsettings.json` y variables de entorno

### Autorización

- Cada endpoint valida permisos del usuario actual
- Los datos devueltos respetan la tenencia (multi-tenancy)
- Las herramientas solo retornan datos que el usuario puede ver

**Ejemplo:** Si accedes como `tenant-A`, solo ves workflows de `tenant-A`.

### Claves de Seguridad

Para producción, **SIEMPRE** configura:
- `Elsa__Identity__SigningKey` - Mínimo 256 bits para JWT
- `Backend__ApiKey` - Clave API para autenticación
- `Cors__AllowedOrigins` - Orígenes específicos (no usar `*`)

Consulta [`CONFIGURATION.md`](src/Elsa.Copilot.Workbench/CONFIGURATION.md) para más detalles.

---

## 🚀 Próximos Pasos para Desarrolladores

1. **Compilar y ejecutar:**
   ```bash
   dotnet run --project src/Elsa.Copilot.Workbench/Elsa.Copilot.Workbench.csproj
   ```

2. **Probar el endpoint de chat:**
   ```bash
   curl -X POST https://localhost:7224/copilot/chat \
     -H "Content-Type: application/json" \
     -H "Authorization: ApiKey [admin-key]" \
     -d '{"message":"Hola, ¿qué actividades hay disponibles?"}'
   ```

3. **Acceder a Elsa Studio:**
   Abre `https://localhost:7224` en el navegador

4. **Revisar los módulos:**
   Explora `src/Modules/Core/Elsa.Copilot.Modules.Core.Chat/` para entender la implementación

---

## 📚 Recursos Útiles

- **Documentación de Elsa:** [elsa-workflows.github.io](https://elsa-workflows.github.io)
- **GitHub Copilot SDK Docs:** GitHub Docs
- **ASP.NET Core:** [microsoft.com/dotnet](https://microsoft.com/dotnet)
- **Blazor:** [microsoft.com/blazor](https://microsoft.com/blazor)
- **Configuración del Proyecto:** [`CONFIGURATION.md`](src/Elsa.Copilot.Workbench/CONFIGURATION.md)

---

## ❓ Preguntas Frecuentes

**P: ¿Necesito GitHub Copilot para usar esto?**
R: Sí, la integración utiliza el SDK de GitHub Copilot. Necesitarás configurar las credenciales en variables de entorno.

**P: ¿Puedo usar otro proveedor de IA?**
R: Actualmente no. El proyecto está diseñado específicamente para GitHub Copilot SDK. Futuras versiones podrían abstraer esto.

**P: ¿Es obligatorio usar el módulo de chat?**
R: No. El Chat Module es opcional. Puedes remover `svc.AddCopilotChat()` en `ModuleRegistration.cs` y el proyecto seguirá funcionando.

**P: ¿Cómo modifico un workflow con IA?**
R: En Fase 2, la IA propondrá cambios que revisar. Actualmente solo responde preguntas (Fase 1).

**P: ¿Dónde configuro las variables de entorno?**
R: Consulta [`src/Elsa.Copilot.Workbench/CONFIGURATION.md`](src/Elsa.Copilot.Workbench/CONFIGURATION.md) para instrucciones detalladas.

---

## 📞 Contacto y Contribuciones

Para reportar bugs o sugerir mejoras:
- **GitHub Issues:** [github.com/elsa-workflows/elsa-copilot/issues](https://github.com/elsa-workflows/elsa-copilot/issues)
- **Discussiones:** [Comunidad Elsa](https://github.com/orgs/elsa-workflows/discussions)

---

**Última actualización:** Marzo 2026  
**Estado del Proyecto:** En desarrollo activo (Fase 1) 
**Licencia:** Revisa LICENSE.md en el repositorio
