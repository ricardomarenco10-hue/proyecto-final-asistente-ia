# PROYECTO FINAL --- Asistente de consultas comerciales con IA

**Curso:** AI Automation --- Coderhouse\
**Responsable:** Ricardo Marenco\
**Estado:** Prototipo académico funcional, con manejo de errores
configurado y validación final pendiente.\
**Stack:** n8n local · Ollama · Qwen 2.5 3B · Google Sheets · Webhook
HTTP POST.

## 1. Resumen ejecutivo

El proyecto automatiza una primera etapa de atención de consultas
comerciales del sector farmacéutico. Recibe una consulta mediante un
Webhook HTTP, recupera el registro/contexto disponible en Google Sheets,
procesa la información mediante un AI Agent conectado a Qwen 2.5 3B
ejecutado localmente con Ollama y actualiza la fila correspondiente en
Google Sheets.

El prototipo simula la recepción de mensajes tipo WhatsApp. **No utiliza
un número real ni la API de WhatsApp Business, y no envía mensajes a
WhatsApp.**

En la sesión de corrección se verificaron individualmente: - Webhook:
recibió una petición POST con `body.consulta_original`. - Google Sheets
--- Get row(s): recuperó una fila existente. - AI Agent: ejecutó el
modelo local y produjo una clasificación y respuesta. - Google Sheets
--- Update row: ejecutó correctamente la actualización de la fila.

Se creó un workflow independiente con `Error Trigger` y se seleccionó
como **Error Workflow** del workflow principal en Settings. La prueba de
disparo ante un fallo real y cualquier notificación/registro adicional
deben validarse antes de afirmar que fueron probados.

## 2. Problema y caso de negocio

Las consultas comerciales requieren lectura, interpretación, preparación
de una primera respuesta y registro para seguimiento. La gestión manual
consume tiempo y puede producir demoras o registros incompletos.

El prototipo busca: - Reducir el tiempo de preparación de respuestas
iniciales. - Estandarizar la clasificación y el registro. - Mantener
trazabilidad básica en una hoja. - Identificar consultas que requieren
revisión humana.

La automatización es de apoyo: no sustituye la validación humana ni los
sistemas comerciales oficiales. No debe generar compromisos sobre
precio, stock, disponibilidad o fechas sin información validada.

## 3. Arquitectura y fuentes de entrada

### Secuencia implementada

`Webhook (POST) → Google Sheets: Get row(s) in sheet → AI Agent (Ollama Chat Model) → Google Sheets: Update row in sheet`

  -----------------------------------------------------------------------
  Componente              Función                 Estado
  ----------------------- ----------------------- -----------------------
  Webhook HTTP POST       Recibe el evento de     Probado
                          prueba y el campo       
                          `consulta_original`     

  Google Sheets --- Get   Recupera la             Probado
  row(s)                  fila/contexto de la     
                          hoja                    

  AI Agent                Clasifica y redacta una Probado
                          salida breve            

  Ollama --- Qwen 2.5 3B  Modelo local conectado  Probado en ejecución
                          al agente               

  Google Sheets ---       Actualiza el registro   Probado
  Update row              existente               

  Workflow de errores --- Recibe errores del      Configurado; falta
  Error Trigger           workflow principal      prueba de fallo real
                          mediante la             
                          configuración Error     
                          Workflow                
  -----------------------------------------------------------------------

**Fuentes de entrada distintas:** (1) Webhook HTTP POST y (2) Google
Sheets como fuente de registros/contexto. Google Sheets también es el
destino de persistencia.

El nodo de actualización utiliza `consulta_original` como columna de
coincidencia. Para un despliegue productivo se recomienda reemplazarla
por un identificador único e inmutable.

## 4. Configuración del AI Agent

### System Message configurado

> Eres un asistente de inteligencia artificial especializado en
> consultas comerciales del sector farmacéutico en Uruguay.\
> Tu función es analizar consultas, clasificarlas y generar respuestas
> breves y profesionales.\
> Clasifica cada consulta en una categoría: Pedido, Entrega, Producto,
> Precio, Documentación, Consulta comercial u Otra.\
> Asigna prioridad Alta, Media o Baja.\
> Determina si requiere revisión humana.\
> No inventes precios, stock, fechas de entrega, disponibilidad ni
> compromisos comerciales.\
> Si falta información o la consulta requiere validación comercial,
> indica que necesita revisión humana.\
> Responde siempre en español profesional, con claridad y precisión.

### Prompt de usuario

El prompt de usuario conserva la consulta recuperada desde Google Sheets
mediante la expresión:

`{{ $json.consulta_original }}`

El agente genera una salida estructurada en texto con categoría,
resumen, prioridad y decisión de revisión humana. En las pruebas
observadas, la salida del modelo se presenta como texto; no se ha
validado un parser de salida JSON estricto.

### Por qué AI Agent y no una cadena LLM simple

El agente se utiliza para interpretar el contenido, clasificarlo y tomar
una decisión operativa ---si requiere o no revisión humana--- dentro de
límites explícitos. La salida alimenta la actualización del registro.
Para una versión más avanzada, se podrían incorporar herramientas y
reglas de derivación; no se afirma que esas herramientas estén
implementadas en esta versión.

## 5. Elección del modelo: local frente a nube

Se seleccionó **Qwen 2.5 3B mediante Ollama local**.

-   **Privacidad:** el procesamiento del modelo se realiza en el equipo
    local, reduciendo la necesidad de enviar el texto de las consultas a
    un proveedor externo de inferencia. Google Sheets sigue siendo un
    servicio externo y requiere proteger sus permisos.
-   **Costo:** no hay costo por token de una API comercial para la
    inferencia local. Sí existen costos de hardware, energía,
    mantenimiento y disponibilidad del equipo.
-   **Independencia:** permite ejecutar el modelo sin depender de la
    disponibilidad de una API de IA en la nube.
-   **Rendimiento y límites:** un modelo 3B local puede tener menor
    capacidad y velocidad que modelos cloud de mayor tamaño; la latencia
    depende del hardware disponible. Se requiere medirla con un volumen
    real antes de producción.

## 6. Manejo de errores y robustez

Se creó el workflow independiente `GESTIÓN DE ERRORES – ASISTENTE IA`,
con un nodo `Error Trigger`. Luego se seleccionó ese workflow en el
campo **Error Workflow** de Settings del workflow principal.

Escenarios contemplados: - **Ollama o equipo local no disponible:** el
agente puede no responder; registrar el error, reintentar de forma
controlada y derivar a una persona. - **Google Sheets/API o credenciales
no disponibles:** no se podrá leer o persistir el resultado; revisar
autorización y evitar perder trazabilidad. - **Consulta ambigua o
insuficiente:** solicitar aclaración o marcar revisión humana. -
**Respuesta fuera de alcance:** no generar compromisos comerciales y
derivar para validación.

**Estado de validación:** el Error Trigger y su asociación al workflow
principal quedaron configurados. No se debe afirmar que el disparo por
fallo real, reintentos, notificaciones o persistencia del error fueron
probados hasta realizar esa prueba y agregar las acciones
correspondientes al workflow de errores.

## 7. Pruebas y evidencias

### Pruebas observadas en la sesión de corrección

  -----------------------------------------------------------------------
  Prueba                              Resultado observado
  ----------------------------------- -----------------------------------
  POST al Webhook con                 Recibido
  `consulta_original`                 

  Lectura de Google Sheets            Correcta tras reconectar OAuth

  Ejecución del AI Agent con Qwen 2.5 Correcta
  3B                                  

  Actualización de fila en Google     Nodo ejecutado correctamente
  Sheets                              

  System Message                      Incorporado y luego ejecutado
                                      correctamente

  Error Workflow                      Seleccionado en Settings; falta
                                      provocar un fallo real para
                                      comprobar el disparo
  -----------------------------------------------------------------------

### Capturas que deben incorporarse al repositorio

Guardar en una carpeta `capturas/`: 1. Canvas del workflow principal
completo. 2. Ejecución integral con los cuatro nodos en verde. 3.
Configuración del AI Agent mostrando System Message (sin credenciales).
4. Salida del AI Agent. 5. Google Sheets mostrando `respuesta_ia`
actualizada. 6. Settings del workflow principal mostrando el Error
Workflow seleccionado. 7. Ejecución de prueba del workflow de errores,
una vez provocado y validado un fallo controlado.

Antes de compartir capturas, ocultar correos, IDs privados de hojas,
tokens, secretos y datos reales.

## 8. Caso de negocio y ROI estimado

Los siguientes valores son **supuestos ilustrativos de modelización**,
no mediciones operativas confirmadas:

  Variable                                              Supuesto
  ---------------------------------- ---------------------------
  Consultas por día                                           15
  Días hábiles por mes                                        22
  Consultas mensuales                                        330
  Tiempo manual por consulta                           5 minutos
  Tiempo manual mensual                               27,5 horas
  Reducción estimada del tiempo                              60%
  Horas ahorradas por mes                             16,5 horas
  Valor de hora de trabajo                                USD 10
  Beneficio bruto mensual estimado                       USD 165
  Mantenimiento mensual estimado       2 horas × USD 10 = USD 20
  Beneficio neto mensual estimado                    **USD 145**

### Fórmulas

-   Consultas mensuales = 15 × 22 = 330.
-   Horas manuales = 330 × 5 / 60 = 27,5.
-   Horas ahorradas = 27,5 × 60% = 16,5.
-   Beneficio bruto = 16,5 × USD 10 = USD 165.
-   Beneficio neto mensual = USD 165 − USD 20 = **USD 145**.

No se informó el costo inicial de desarrollo/implementación; por lo
tanto, no corresponde declarar un porcentaje definitivo de ROI ni un
período de recupero como si fueran datos observados.

Si `C` es el costo inicial: - Recupero estimado en meses = `C / 145`. -
ROI a 12 meses = `((145 × 12 − C) / C) × 100`.

El ROI real debe recalcularse con volumen, tiempo ahorrado, costo
laboral, mantenimiento y costo inicial medidos. La automatización pierde
conveniencia si el volumen es bajo, la tasa de excepciones es alta o la
revisión humana consume la mayor parte del tiempo ahorrado.

## 9. Privacidad, límites y despliegue

-   Utilizar datos ficticios o anonimizados en pruebas.
-   No incluir datos personales, clínicos o comerciales confidenciales
    en repositorios públicos.
-   No exportar credenciales OAuth, client secrets ni tokens en el JSON
    del workflow.
-   El entorno local `http://localhost:5678` no es accesible desde
    Internet por sí solo.
-   El prototipo depende del equipo local y no ofrece disponibilidad
    24/7.
-   No está integrada la API oficial de WhatsApp Business ni existe
    envío real de mensajes.

## 10. Entregables y checklist final

-   [ ] Exportar el workflow principal desde n8n como JSON, sin
    credenciales.
-   [ ] Exportar el workflow de errores como JSON, sin credenciales.
-   [ ] Adjuntar este README en el repositorio.
-   [ ] Incorporar capturas reales en `capturas/`.
-   [ ] Probar un fallo controlado y guardar evidencia del Error
    Trigger.
-   [ ] Confirmar que la hoja conserva la salida actualizada.
-   [ ] Completar el costo inicial para calcular ROI porcentual y
    payback.
-   [ ] Revisar que no se hayan incluido datos ni secretos sensibles.

## 11. Conclusión

Se recuperó y verificó el circuito principal del prototipo: recepción
HTTP, lectura de Google Sheets, análisis mediante AI Agent con Qwen 2.5
3B local y actualización de la hoja. Se incorporó un System Message y se
configuró un workflow independiente de Error Trigger como manejador de
errores del workflow principal.

El alcance sigue siendo un prototipo académico que simula la recepción
de mensajes de WhatsApp. Antes de considerarlo listo para producción o
declarar la robustez plenamente probada, deben completarse la prueba de
fallo real, las evidencias, la exportación de workflows y la medición
del ROI con costos reales.
