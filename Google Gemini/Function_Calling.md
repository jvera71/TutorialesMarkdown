¡Claro que sí! Aquí tienes un tutorial detallado sobre la funcionalidad de **Function Calling** de Google Gemini, utilizando el código C# proporcionado como base y centrándonos en el español de España.

---

## Tutorial Avanzado: Function Calling en Google Gemini

En este tutorial, vamos a explorar una de las funcionalidades más potentes y transformadoras de los modelos de Gemini: el **Function Calling**. No nos limitaremos a una explicación básica, sino que profundizaremos en su propósito, su flujo de trabajo y, lo más importante, cómo se integra con otras capacidades de Gemini para crear soluciones verdaderamente avanzadas.

### ¿Qué es y para qué sirve el Function Calling?

Imagina que Google Gemini es un director de orquesta increíblemente inteligente. Puede leer y entender partituras complejas (tus prompts), pero no puede tocar los instrumentos por sí mismo. El **Function Calling** es la batuta que le permite a Gemini dirigir a los músicos (tu código, tus APIs, tus bases de datos) para que ejecuten acciones concretas en el mundo real.

En esencia, el Function Calling permite al modelo Gemini **convertir el lenguaje natural en llamadas a funciones ejecutables**. En lugar de generar simplemente texto, el modelo puede identificar cuándo una petición de un usuario requiere una acción externa y responder con una estructura de datos (JSON) que especifica qué función de tu código se debe llamar y con qué argumentos.

Esto transforma a Gemini de un mero generador de contenido a un **orquestador de tareas**.

**Casos de uso clave:**

*   **Integración con APIs externas:** Obtener datos en tiempo real (el tiempo, precios de acciones, estado de un vuelo).
*   **Acciones en sistemas internos:** Crear un ticket en Jira, añadir un cliente a un CRM (Salesforce, HubSpot), actualizar el inventario en un ERP.
*   **Control de dispositivos (IoT):** "Enciende las luces del salón y pon la temperatura a 22 grados".
*   **Consultas a bases de datos:** Traducir una pregunta como "¿Cuántos usuarios se registraron la semana pasada?" en una consulta SQL o una llamada a un ORM.

### El Flujo de Trabajo: Un Baile en Dos Pasos

El proceso de Function Calling no es una única llamada, sino una conversación estructurada entre tu aplicación y el modelo.

1.  **Primer Paso: La Petición y la Sugerencia**
    *   **Tú (el desarrollador):** Envías un *prompt* a Gemini. Junto con el prompt, proporcionas una lista de "herramientas" (`Tools`). Cada herramienta contiene una o más declaraciones de funciones (`FunctionDeclarations`), que son como un contrato: le dices al modelo qué funciones existen, qué hacen (con una buena descripción) y qué parámetros aceptan.
    *   **Gemini (el modelo):** Analiza el prompt. Si determina que la mejor manera de responder es usando una de tus funciones, no te devolverá una respuesta en texto. En su lugar, te devolverá un objeto `FunctionCall` que contiene:
        *   `Name`: El nombre de la función que cree que debe ser llamada.
        *   `Args`: Un objeto JSON con los argumentos que ha extraído del prompt del usuario.

2.  **Segundo Paso: La Ejecución y la Respuesta Final**
    *   **Tú (el desarrollador):** Recibes este `FunctionCall`. Tu código es ahora responsable de:
        a. Identificar la función solicitada (ej. `find_theaters`).
        b. Ejecutar esa función real en tu sistema (llamar a tu API, consultar tu base de datos, etc.) usando los argumentos proporcionados.
        c. Capturar el resultado de esa ejecución (por ejemplo, una lista de cines).
    *   **Tú (de nuevo):** Vuelves a llamar a Gemini. Esta vez, le envías el historial de la conversación, incluyendo el `FunctionCall` original del modelo y un nuevo objeto `FunctionResponse` que contiene el resultado que obtuviste al ejecutar la función.
    *   **Gemini (el modelo):** Recibe este nuevo contexto (el resultado de la función) y ahora sí, genera una respuesta final en lenguaje natural para el usuario, basada en los datos reales que le has proporcionado.

### Ejemplo Práctico en C#

El siguiente código, extraído de los tests de la librería, ilustra perfectamente el **primer paso** de este flujo. Le preguntamos al modelo por cines que proyecten una película y le proporcionamos un conjunto de herramientas que puede utilizar.

```csharp
[Fact]
// Ref: https://ai.google.dev/docs/function_calling
public async Task Function_Calling()
{
    // Arrange
    var prompt = "Which theaters in Mountain View show Barbie movie?";
    var googleAi = new GoogleAI(apiKey: _fixture.ApiKey);
    var model = _googleAi.GenerativeModel(model: _model);
    List<Tool> tools =
    [
        new Tool()
        {
            FunctionDeclarations =
            [
                new()
                {
                    Name = "find_movies",
                    Description =
                        "find movie titles currently playing in theaters based on any description, genre, title words, etc.",
                    Parameters = new()
                    {
                        Type = ParameterType.Object,
                        Properties = new
                        {
                            Location = new
                            {
                                Type = ParameterType.String,
                                Description =
                                    "The city and state, e.g. San Francisco, CA or a zip code e.g. 95616"
                            },
                            Description = new
                            {
                                Type = ParameterType.String,
                                Description =
                                    "Any kind of description including category or genre, title words, attributes, etc."
                            }
                        },
                        Required = ["description"]
                    }
                },


                new()
                {
                    Name = "find_theaters",
                    Description =
                        "find theaters based on location and optionally movie title which are is currently playing in theaters",
                    Parameters = new()
                    {
                        Type = ParameterType.Object,
                        Properties = new
                        {
                            Location = new
                            {
                                Type = ParameterType.String,
                                Description =
                                    "The city and state, e.g. San Francisco, CA or a zip code e.g. 95616"
                            },
                            Movie = new { Type = ParameterType.String, Description = "Any movie title" }
                        },
                        Required = ["location"]
                    }
                },


                new()
                {
                    Name = "get_showtimes",
                    Description = "Find the start times for movies playing in a specific theater",
                    Parameters = new()
                    {
                        Type = ParameterType.Object,
                        Properties = new
                        {
                            Location = new
                            {
                                Type = ParameterType.String,
                                Description =
                                    "The city and state, e.g. San Francisco, CA or a zip code e.g. 95616"
                            },
                            Movie = new { Type = ParameterType.String, Description = "Any movie title" },
                            Theater = new { Type = ParameterType.String, Description = "Name of the theater" },
                            Date = new
                            {
                                Type = ParameterType.String, Description = "Date for requested showtime"
                            }
                        },
                        Required = ["location", "movie", "theater", "date"]
                    }
                }
            ]
        }
    ];

    // Act
    var response = await model.GenerateContent(prompt, tools: tools);

    // Assert
    response.Should().NotBeNull();
    response.Candidates.Should().NotBeNull().And.HaveCount(1);
    response?.Candidates?[0]?.Content?.Parts[0]?.FunctionCall?.Should().NotBeNull();
    _output.WriteLine(response?.Candidates?[0]?.Content?.Parts[0]?.FunctionCall?.Name);
    _output.WriteLine(response?.Candidates?[0]?.Content?.Parts[0]?.FunctionCall?.Args?.ToString());
}
```

**Análisis del Ejemplo:**

*   **El Prompt:** Es una pregunta en lenguaje natural: `"Which theaters in Mountain View show Barbie movie?"`.
*   **Las Herramientas (`Tools`):** Le estamos dando a Gemini un "catálogo" de capacidades. Fíjate en la importancia de las `Description`. Así es como el modelo sabe qué herramienta usar para qué propósito.
*   **La Respuesta del Modelo (esperada):** El modelo no responderá "El cine AMC lo proyecta". En su lugar, devolverá un `FunctionCall` con `Name: "find_theaters"` y `Args: { "Location": "Mountain view, CA", "Movie": "Barbie" }`.
*   **Tu Tarea Siguiente:** Tu código tomaría esta respuesta, llamaría a tu API de cines con esos parámetros, y luego continuaría la conversación con Gemini para dar la respuesta final al usuario.

### Interacciones Avanzadas con Otras Funcionalidades de Gemini

Aquí es donde el Function Calling realmente brilla, al combinarlo con otras capacidades.

#### 1. Function Calling + Análisis Multimodal (`Analyze_Document_PDF_From_FileAPI`)

Imagina un sistema de contabilidad automatizado.

*   **Escenario:** Un gestor sube una factura en PDF a un chat y escribe: "Procesa esta factura y añádela a nuestro sistema de contabilidad para pagarla el mes que viene".
*   **Flujo:**
    1.  Tu aplicación utiliza `Analyze_Document_PDF_From_FileAPI` para que Gemini procese el PDF. El prompt inicial ya incluye la petición del usuario.
    2.  Le proporcionas una herramienta con una función `crear_registro_factura(proveedor: string, importe_total: float, fecha_vencimiento: date)`.
    3.  Gemini lee el PDF, extrae el nombre del proveedor, el importe total y la fecha de la factura.
    4.  A continuación, calcula la "fecha de vencimiento" basándose en "pagarla el mes que viene".
    5.  Gemini responde con un `FunctionCall` a `crear_registro_factura` con los datos extraídos y calculados.
    6.  Tu aplicación ejecuta esta función, que inserta los datos en tu ERP o base de datos de contabilidad.
    7.  Finalmente, tu aplicación responde al gestor en el chat: "Factura de [Proveedor] por [Importe] registrada correctamente con vencimiento el [Fecha]".

#### 2. Function Calling + Búsqueda en Google (`Generate_Content_with_Google_Search`)

Creemos un agente de análisis de mercado.

*   **Escenario:** Un analista financiero pregunta: "¿Cuál es el sentimiento general de las noticias sobre las acciones de ACME Corp. en las últimas 24 horas y, si es positivo, compra 100 acciones?".
*   **Flujo:**
    1.  Tu aplicación recibe el prompt y se lo pasa a Gemini, activando la herramienta `GoogleSearchRetrieval`.
    2.  Gemini utiliza la búsqueda de Google para encontrar los artículos de noticias más recientes sobre "ACME Corp.".
    3.  Le has proporcionado dos funciones: `analizar_sentimiento(texto: string)` y `ejecutar_orden_compra(ticker: string, cantidad: int)`.
    4.  Gemini recibe los resultados de la búsqueda y, en una segunda vuelta (o en modo multiturno), genera un `FunctionCall` a `analizar_sentimiento` con el contenido de las noticias.
    5.  Tu aplicación ejecuta el análisis (que podría ser otra llamada a un modelo de IA o una librería local) y devuelve, por ejemplo, `{"sentimiento": "positivo", "confianza": 0.85}`.
    6.  Envías este resultado a Gemini. El modelo, siguiendo la lógica original del prompt ("si es positivo, compra"), ahora genera un segundo `FunctionCall`: `ejecutar_orden_compra("ACME", 100)`.
    7.  Tu aplicación se conecta a la API de tu bróker y ejecuta la orden.
    8.  Respondes al analista: "El sentimiento es positivo. Se ha ejecutado una orden de compra de 100 acciones de ACME.".

#### 3. Function Calling + Chat Multimodal (`Start_Chat_With_Multimodal_Content`)

Vamos a crear un asistente de soporte técnico para una fábrica.

*   **Escenario:** Un operario hace una foto a una pieza rota de una máquina, la sube a una app de chat y escribe: "Esta pieza se ha roto. ¿Tenemos recambios? Si no, pídela con urgencia".
*   **Flujo:**
    1.  El `ChatSession` se inicia con el prompt y la imagen.
    2.  Le has proporcionado las funciones `consultar_inventario(id_pieza: string)` y `crear_pedido_urgente(id_pieza: string, cantidad: int)`.
    3.  Gemini analiza la imagen, identifica la pieza y su número de serie (ej: "Filtro-H2O-X5").
    4.  Genera un `FunctionCall` a `consultar_inventario("Filtro-H2O-X5")`.
    5.  Tu sistema consulta la base de datos de inventario y devuelve `{"stock": 0}`.
    6.  Le pasas esta respuesta a Gemini. El modelo ve que el stock es cero y, siguiendo la instrucción original, genera un `FunctionCall` a `crear_pedido_urgente("Filtro-H2O-X5", 1)`.
    7.  Tu sistema crea el pedido en el sistema de compras.
    8.  El chat responde al operario: "Pieza 'Filtro-H2O-X5' identificada. No hay stock disponible. Se ha creado un pedido urgente con número #P8921."

### Conclusión

El **Function Calling** es mucho más que una simple característica; es un cambio de paradigma. Permite que los modelos de lenguaje dejen de ser un oráculo de información para convertirse en el cerebro de una aplicación, capaz de razonar, delegar tareas y actuar sobre el mundo digital y físico a través de tu código. Al combinarlo de forma creativa con las capacidades multimodales, de búsqueda y de chat de Gemini, las posibilidades para crear agentes autónomos y asistentes inteligentes son prácticamente ilimitadas.