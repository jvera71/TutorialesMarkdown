Claro, aquí tienes un tutorial en español de España sobre la funcionalidad `Function Calling` de Gemini, centrado en el escenario avanzado de conversaciones multi-turno con múltiples llamadas a funciones.

---

# Tutorial Avanzado de Gemini: `Function Calling` en Conversaciones Multi-Turn

Google Gemini no es solo un modelo de lenguaje capaz de generar texto; es una herramienta potente que puede interactuar con sistemas externos para realizar tareas complejas. Una de sus funcionalidades más avanzadas es el **Function Calling** (llamada a funciones), que permite al modelo delegar tareas en tu propio código.

Este tutorial se centra en un caso de uso avanzado: **mantener una conversación fluida (multi-turno) donde el modelo necesita realizar varias llamadas a diferentes funciones** para satisfacer las peticiones consecutivas de un usuario. Usaremos como referencia el ejemplo `Function_Calling_MultiTurn_Multiple`.

## ¿Para qué sirve esta funcionalidad?

Imagina que estás construyendo un asistente virtual. Un usuario no suele hacer una única pregunta aislada. Lo normal es que mantenga una conversación, donde una pregunta lleva a la siguiente.

La funcionalidad `Function_Calling_MultiTurn_Multiple` permite a Gemini gestionar este tipo de diálogos complejos. Su propósito es:

1.  **Mantener el Contexto:** El modelo recuerda los intercambios anteriores en la conversación (peticiones del usuario, respuestas del modelo y resultados de funciones).
2.  **Identificar la Tarea Correcta:** Basándose en el contexto, Gemini es capaz de entender que una nueva pregunta del usuario requiere una herramienta (función) diferente a la que usó en el paso anterior.
3.  **Orquestar Tareas Secuenciales:** Permite crear flujos de trabajo donde el resultado de una tarea informa la siguiente. Por ejemplo, primero buscar un producto y, a continuación, añadirlo a la cesta de la compra.

En esencia, esta funcionalidad es la clave para pasar de un simple "contestador de preguntas" a un **agente conversacional proactivo y resolutivo**, capaz de ejecutar una secuencia de acciones en el mundo real (llamar a APIs, consultar bases de datos, etc.).

## Interacciones con otras funcionalidades de Gemini

Para que este flujo funcione, se apoya en varias características de Gemini que trabajan en conjunto:

*   **Historial de Conversación (`Contents`):** Es el pilar fundamental. A diferencia de las llamadas simples, aquí debemos construir manualmente el historial completo de la conversación y enviarlo en cada petición. Este historial incluye cada turno con su rol específico:
    *   `Role.User`: Lo que dice el usuario.
    *   `Role.Model`: La respuesta del modelo, que puede ser texto o una petición para llamar a una función (`FunctionCall`).
    *   `Role.Function`: El resultado que tu código devuelve al modelo tras ejecutar la función solicitada (`FunctionResponse`).
*   **Declaración de Herramientas (`Tools`):** Antes de iniciar la conversación, debes proporcionar a Gemini una lista de todas las "herramientas" (funciones) que tiene a su disposición. La clave del éxito reside en la **descripción de cada función**. Una descripción clara y precisa es lo que permite al modelo decidir qué herramienta es la más adecuada para cada momento de la conversación.
*   **Generación de Contenido (`GenerateContent`):** Aunque se puede usar en un chat, este escenario avanzado se beneficia de la llamada directa a `GenerateContent` sobre el modelo, pasándole un objeto `GenerateContentRequest` que contiene tanto las herramientas disponibles como el historial completo de la conversación.

## Ejemplos de Interacciones Avanzadas

Las posibilidades son enormes. Aquí tienes algunos ejemplos avanzados que ilustran el poder de este enfoque:

*   **Asistente de Planificación de Viajes:**
    1.  **Usuario:** "Busca vuelos a Roma para la primera semana de julio."
    2.  **Gemini -> `find_flights()`:** El modelo detecta la necesidad de buscar vuelos y llama a tu función.
    3.  **Tu código:** Ejecuta la búsqueda y devuelve una lista de vuelos.
    4.  **Gemini:** "He encontrado varios vuelos. El más económico es con Iberia por 150€. ¿Quieres que busque alojamiento?"
    5.  **Usuario:** "Sí, busca hoteles de 4 estrellas cerca del Coliseo para esas fechas."
    6.  **Gemini -> `find_hotels()`:** Usando las fechas del contexto anterior, Gemini ahora llama a una función diferente para buscar hoteles.
    7.  **Tu código:** Ejecuta la búsqueda de hoteles y la devuelve.
    8.  **Gemini:** "Perfecto. El 'Hotel Foro Romano' tiene disponibilidad. ¿Procedo con la reserva de vuelo y hotel?"

*   **Asistente de Soporte Técnico:**
    1.  **Usuario:** "Mi pedido nº 12345 no ha llegado."
    2.  **Gemini -> `get_order_status(12345)`:** El modelo identifica el número de pedido y llama a la función para obtener su estado.
    3.  **Tu código:** Consulta la base de datos y devuelve que el pedido está "en reparto".
    4.  **Gemini:** "Veo que tu pedido está en reparto y la entrega estimada es hoy. ¿Puedo ayudarte en algo más?"
    5.  **Usuario:** "Sí, ¿puedes enviarme el enlace de seguimiento del transportista?"
    6.  **Gemini -> `get_tracking_link(12345)`:** Reutilizando el número de pedido del contexto, el modelo solicita ahora una acción diferente.

## Código de Ejemplo: De la Búsqueda de Cines a la Recomendación de Películas

El siguiente código C# muestra cómo se construye una conversación multi-turno. El usuario primero pregunta por cines que proyectan una película y, en un segundo turno, pide recomendaciones de otro género de películas, forzando al modelo a elegir una función distinta.

```csharp
[Fact]
// Ref: https://ai.google.dev/docs/function_calling#multi-turn-example-2
public async Task Function_Calling_MultiTurn_Multiple()
{
    // Arrange
    // 1. Petición inicial del usuario.
    var prompt = "Which theaters in Mountain View show Barbie movie?";
    var googleAi = new GoogleAI(apiKey: _fixture.ApiKey);
    var model = _googleAi.GenerativeModel(model: _model);
    
    // 2. Definimos las herramientas que el modelo puede usar.
    // La descripción es crucial para que el modelo elija la correcta.
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
                    Parameters = new() { /* ... */ }
                },
                new()
                {
                    Name = "find_theaters",
                    Description =
                        "find theaters based on location and optionally movie title which are is currently playing in theaters",
                    Parameters = new() { /* ... */ }
                },
                new()
                {
                    Name = "get_showtimes",
                    Description = "Find the start times for movies playing in a specific theater",
                    Parameters = new() { /* ... */ }
                }
            ]
        }
    ];

    // 3. Construimos el historial de la conversación manualmente.
    var request = new GenerateContentRequest(prompt, tools: tools);
    request.Contents[0].Role = Role.User;
    
    // TURNO 1: El modelo decide llamar a la función 'find_theaters'.
    // Esto simula la primera respuesta del modelo.
    request.Contents.Add(new Content()
    {
        Role = Role.Model,
        Parts = new()
        {
            new FunctionCall()
            {
                Name = "find_theaters",
                Args = new { Location = "Mountain View, CA", Movie = "Barbie" }
            }
        }
    });

    // TURNO 2: Le proporcionamos el resultado de nuestra función 'find_theaters'.
    // En una aplicación real, aquí llamaríamos a nuestra API o base de datos.
    request.Contents.Add(new Content()
    {
        Role = Role.Function,
        Parts = new()
        {
            new FunctionResponse()
            {
                Name = "find_theaters",
                Response = new
                {
                    Movie = "Barbie",
                    Theaters = new dynamic[]
                    {
                        new { Name = "AMC Mountain View 16", Address = "2000 W El Camino Real..." },
                        new { Name = "Regal Edwards 14", Address = "245 Castro St..." }
                    }
                }
            }
        }
    });

    // TURNO 3: El modelo usa el resultado de la función para responder al usuario en lenguaje natural.
    request.Contents.Add(new Content()
    {
        Role = Role.Model,
        Parts = new()
        {
            new TextData()
            {
                Text = "OK. I found two theaters in Mountain View showing Barbie: AMC Mountain View 16 and Regal Edwards 14."
            }
        }
    });

    // TURNO 4: El usuario hace una nueva pregunta que requiere una herramienta diferente.
    request.Contents.Add(new Content()
    {
        Role = Role.User,
        Parts = new()
        {
            new TextData() { Text = "Can we recommend some comedy movies on show in Mountain View?" }
        }
    });

    // Act
    // Enviamos la conversación completa al modelo para que genere la siguiente acción.
    var response = await model.GenerateContent(request);

    // Assert
    // El resultado esperado es que el modelo ahora solicite llamar a 'find_movies' con 'comedy' como descripción.
    response.Should().NotBeNull();
    response.Candidates.Should().NotBeNull().And.HaveCount(1);
    response?.Candidates?[0]?.Content?.Parts[0]?.FunctionCall?.Should().NotBeNull();
    _output.WriteLine(response?.Candidates?[0]?.Content?.Parts[0]?.FunctionCall?.Name);
    _output.WriteLine(response?.Candidates?[0]?.Content?.Parts[0]?.FunctionCall?.Args?.ToString());
}
```

### Conclusión

Dominar el flujo de `Function_Calling_MultiTurn_Multiple` es esencial para construir aplicaciones de IA verdaderamente interactivas y útiles con Gemini. Al gestionar el historial de la conversación y proporcionar un conjunto bien definido de herramientas, puedes capacitar al modelo para que actúe como un orquestador inteligente, resolviendo problemas complejos que requieren múltiples pasos y acceso a sistemas externos.