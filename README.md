# RPA Challenge + Orchestrator Queue (FinCorp_Invoices)

## Overview
Solución RPA en UiPath con arquitectura Dispatcher/Performer.
- **Dispatcher**: Lee `Input_Invoices.xlsx` y carga transacciones en la cola `FinCorp_Invoices` en Orchestrator (usando Reference como clave).
- **Performer (REFramework)**: Consume la cola y completa el formulario del sitio RPA Challenge. Además ejecuta una “caja negra” que devuelve un objeto tipo JSON/Dictionary con `finalAmount`, `taxApplied`, `status`, `normalizedDate` y decide si enviar o no a la web.

## Requisitos
- UiPath Studio (Modern)
- Acceso a Orchestrator (cola: `FinCorp_Invoices`)
- Navegador Edge/Chrome
- URL del reto: https://rpachallenge.com/

## Cómo ejecutar
### 1) Dispatcher
1. Abrir el proyecto Dispatcher
2. Configurar Orchestrator (Folder/Workspace) y QueueName = `FinCorp_Invoices`
3. Ejecutar `Main.xaml`
4. Verificar en Orchestrator que se crearon items en estado **New**

### 2) Performer (REFramework)
1. Abrir el proyecto Performer (REFramework)
2. En `Data\Config.xlsx`:
   - `OrchestratorQueueName` = `FinCorp_Invoices`
   - `RpaChallengeUrl` = `https://rpachallenge.com/`
3. Ejecutar `Main.xaml`
4. Verificar ejecución en Orchestrator (Successful/Business Exception)

---

# Decisiones de Diseño (ADR)

## Patrón de diseño elegido
**REFramework (Robotic Enterprise Framework)** para el Performer.
**Razón:** permite escalabilidad y operación en producción por:
- Separación clara entre Inicialización, Obtención de Transacción, Proceso y Cierre
- Reintentos automáticos y logging estructurado
- Integración nativa con Orchestrator Queues (transacciones, estados, excepciones)
Esto se alinea con el requerimiento de escalabilidad planteado.

## Resiliencia de selectores en web dinámica
El formulario reordena los campos en cada ronda. Para evitar dependencia de posición:
- Estrategia: **Click en el label (texto del campo) → TAB → Type Into**.
- Esto ancla cada ingreso al **label** y no al input por ubicación.
- Se usa `Use Application/Browser` en modo **Attach** para reutilizar el navegador abierto.

## Estrategia de manejo de errores (Global vs Local)
- **Global (REFramework):** errores no manejados escalan al framework como System Exception para reintentos/registro.
- **Local (en Process.xaml):**
  - Validación “caja negra”: si `status = REVIEW` (fin de semana), se lanza `BusinessRuleException` para registrar condición de negocio y no enviar a la web.
  - Detección de fin del reto: al detectar “Congratulations”, se lanza `BusinessRuleException` para finalizar de forma controlada y dejar evidencia en logs.

## Caja negra (JSON/Dictionary)
Por cada transacción se construye un objeto `Dictionary(Of String, Object)` con:
- `finalAmount`
- `taxApplied`
- `status` = OK / REVIEW (REVIEW si es fin de semana basado en `normalizedDate`)
- `normalizedDate` en formato `yyyy-MM-dd`
El robot **lee** `status` para decidir si envía el formulario.

---

## Evidencia
- Logs muestran el JSON generado: `BlackBox JSON: {...}`
- Orchestrator muestra transacciones Successful y Business Exception (REVIEW/Congratulations).
