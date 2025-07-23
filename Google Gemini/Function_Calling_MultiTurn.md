¡Claro! Aquí tienes un tutorial en español de España sobre la funcionalidad `Function_Calling_MultiTurn` de Google Gemini, utilizando como referencia el código que has proporcionado.

---

# Tutorial: Function Calling Multi-Turn con Google Gemini

El `Function Calling` (o llamada a funciones) es una de las características más potentes de los modelos de Gemini. Permite que el modelo de IA no solo genere texto, sino que también interactúe con sistemas y APIs externas. El `Function Calling Multi-Turn` lleva esta capacidad un paso más allá, permitiendo conversaciones complejas donde el modelo puede ejecutar una secuencia de herramientas, procesar los resultados y decidir los siguientes pasos de forma autónoma.

## ¿Qué es el Function Calling Multi-Turn?

Imagina que estás manteniendo una conversación con un asistente de IA. En lugar de que la conversación se limite a una sola pregunta y una sola respuesta, el asistente puede realizar varias acciones en tu nombre, una tras otra, manteniendo el contexto de la conversación.

*   **Single-Turn (Un solo turno):** Haces una pregunta, el modelo identifica que necesita una herramienta, la "llama", obtiene un resultado y te da una respuesta final. Fin de la interacción.
*   **Multi-Turn (Múltiples turnos):** Haces una pregunta. El modelo llama a una herramienta. Con el resultado de esa primera herramienta, decide que necesita llamar a una *segunda* herramienta o quizás pedirte más información antes de continuar. Este proceso puede repetirse varias veces dentro de la misma conversación, creando un flujo de trabajo dinámico.

En esencia, el **Multi-Turn** transforma a Gemini de un simple contestador a un agente proactivo capaz de resolver problemas complejos que requieren múltiples pasos.

## ¿Para qué sirve esta funcionalidad?

La capacidad de mantener un diálogo de múltiples turnos con llamadas a funciones abre un abanico de posibilidades para crear aplicaciones avanzadas:

*   **Workflows Complejos:** Automatizar procesos que requieren varias etapas, como planificar un viaje completo: buscar vuelos, luego, con las fechas del vuelo, buscar hoteles y, finalmente, con la ubicación del hotel, alquilar un coche.
*   **Asistentes de Diagnóstico Interactivo:** Un sistema podría guiar a un técnico a través de la resolución de un problema. "Revisa el estado del servidor A" -> (Respuesta de la API: "OK") -> "Vale, entonces comprueba la latencia de la red con el servidor B".
*   **Gestión de Datos en Sistemas Internos:** Crear un nuevo cliente en un CRM, y acto seguido, asignarle una tarea de seguimiento o enviarle un correo de bienvenida a través de otra API.
*   **Investigación y Análisis en Profundidad:** Pedirle al modelo que encuentre artículos sobre un tema, que luego los resuma, y finalmente que extraiga los nombres de los autores y busque sus publicaciones más recientes.

## Interacciones con otras funcionalidades de Gemini

El `Function_Calling_MultiTurn` no es una función aislada, sino que se integra y depende de otras capacidades del ecosistema de Gemini:

*   **`Start_Chat` y el historial de conversación:** La base del "Multi-Turn" es la conversación continua. El historial (`History`) del chat es crucial, ya que proporciona al modelo todo el contexto de los turnos anteriores (tus preguntas, las llamadas a funciones que ha hecho y los resultados que ha obtenido) para que pueda tomar decisiones informadas sobre el siguiente paso.
*   **`Function_Calling` (Single-Turn):** El `Multi-Turn` es una evolución natural del `Function_Calling` básico. Primero se debe dominar la definición y llamada de una sola función para luego poder orquestar varias.
*   **Definición de `Tools` (Herramientas):** Antes de que el modelo pueda llamar a cualquier función, debes declararle qué "herramientas" tiene a su disposición (`FunctionDeclarations`). En estas declaraciones defines el nombre de la función, su descripción (muy importante, para que el modelo sepa cuándo usarla) y los parámetros que necesita.
*   **`Generate_Content`:** Es el motor subyacente que procesa cada turno de la conversación. En un flujo multi-turno, llamarás a `GenerateContent` repetidamente, cada vez con un historial de conversación más extenso que incluye los resultados de las funciones ejecutadas en los pasos anteriores.
*   **`File API` (Analyze_Document_PDF_From_FileAPI, etc.):** Se puede combinar el `Function Calling` con la capacidad de analizar ficheros. Por ejemplo, podrías tener una función `extraer_datos_factura(fileId)` que el modelo decide llamar después de que le pidas "Analiza esta factura en PDF que acabo de subir y crea una entrada en el sistema de contabilidad". El modelo primero procesaría el PDF y luego llamaría a la función con los datos extraídos.

## Ejemplo de Código: Búsqueda de Cines

El siguiente código muestra un ejemplo de cómo se estructura una petición multi-turno. En este escenario, ya ha habido un primer turno donde el modelo ha identificado que necesita buscar cines y la aplicación le ha devuelto el resultado. Ahora, enviamos todo ese historial de vuelta al modelo para que genere una respuesta final en lenguaje natural.

```csharp
[Fact]
// Ref: https://ai.google.dev/docs/function_calling#function-calling-one-and-a-half-turn-curl-sample
public async Task Function_Calling_MultiTurn()
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
                // ... (declaraciones de find_movies, find_theaters, get_showtimes)
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
                // ...
            ]
        }
    ];
    // Se construye una petición que simula una conversación que ya ha comenzado.
    var request = new GenerateContentRequest(prompt, tools: tools);
    request.Contents[0].Role = Role.User; // 1. El usuario pregunta.

    // 2. El modelo, en un turno anterior (no mostrado aquí), decidió llamar a la función `find_theaters`.
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

    // 3. Nuestra aplicación ejecutó la función y ahora le devolvemos el resultado al modelo.
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
                    Name = "find_theaters",
                    Content = new
                    {
                        Movie = "Barbie",
                        Theaters = new dynamic[]
                        {
                            new
                            {
                                Name = "AMC Mountain View 16",
                                Address = "2000 W El Camino Real, Mountain View, CA 94040"
                            },
                            new
                            {
                                Name = "Regal Edwards 14",
                                Address = "245 Castro St, Mountain View, CA 94040"
                            }
                        }
                    }
                }
            }
        }
    });

    // Act
    // 4. Se envía la conversación completa al modelo para que genere la respuesta final.
    var response = await model.GenerateContent(request);

    // Assert
    response.Should().NotBeNull();
    response.Candidates.Should().NotBeNull().And.HaveCount(1);
    response.Text.Should().NotBeEmpty();
    _output.WriteLine(response?.Text); // El modelo responderá algo como: "La película Barbie se proyecta en AMC Mountain View 16 y Regal Edwards 14."
}
```

### Análisis del Flujo del Ejemplo

1.  **Pregunta Inicial del Usuario:** El usuario pregunta por cines que proyecten "Barbie" en Mountain View.
2.  **Primer Turno del Modelo (Simulado):** El modelo recibe la pregunta y las herramientas disponibles. Determina que la herramienta `find_theaters` es la más adecuada y emite una `FunctionCall` con los argumentos `location` y `movie`.
3.  **Ejecución en la Aplicación:** Tu código recibe esta `FunctionCall`, la interpreta y ejecuta la lógica correspondiente (por ejemplo, llamando a una API de cines real). Devuelve una lista de cines.
4.  **Segundo Turno del Modelo:** Ahora construyes una nueva petición. Esta petición contiene **todo el historial**: la pregunta original del usuario, la `FunctionCall` del modelo y la `FunctionResponse` de tu aplicación. Envías todo esto de vuelta a Gemini.
5.  **Respuesta Final:** El modelo, al tener el resultado de la función, ya no necesita llamar a más herramientas. En su lugar, sintetiza la información recibida y genera una respuesta coherente y en lenguaje natural para el usuario.

## Ejemplo Avanzado: Planificación de un Viaje de Negocios

Imagina un asistente de viajes más sofisticado.

1.  **Usuario:** "Necesito organizar un viaje a Madrid para asistir a la 'Conferencia de IA' la próxima semana. Encuentra un vuelo de ida y vuelta y un hotel céntrico para esas fechas."

2.  **Turno 1: Modelo -> `buscar_fechas_conferencia`**
    *   El modelo se da cuenta de que "la próxima semana" es ambiguo y necesita las fechas exactas del evento.
    *   **Llama a la función:** `FunctionCall { Name = "buscar_fechas_conferencia", Args = { Evento = "Conferencia de IA" } }`

3.  **App -> Respuesta de la función**
    *   Tu API busca en una base de datos de eventos.
    *   **Devuelve:** `FunctionResponse { Name = "buscar_fechas_conferencia", Response = { Inicio = "2024-10-21", Fin = "2024-10-23" } }`

4.  **Turno 2: Modelo -> `buscar_vuelos` y `buscar_hoteles` (Llamadas en paralelo)**
    *   El modelo ahora tiene las fechas. Se da cuenta de que puede realizar las dos siguientes tareas a la vez. Los modelos más avanzados de Gemini pueden solicitar múltiples llamadas a funciones en un solo turno.
    *   **Llama a las funciones:**
        *   `FunctionCall { Name = "buscar_vuelos", Args = { Origen = "BCN", Destino = "MAD", Salida = "2024-10-21", Regreso = "2024-10-23" } }`
        *   `FunctionCall { Name = "buscar_hoteles", Args = { Ciudad = "Madrid", CheckIn = "2024-10-21", CheckOut = "2024-10-23", Zona = "Centro" } }`

5.  **App -> Respuestas de las funciones**
    *   Tu aplicación ejecuta ambas búsquedas.
    *   **Devuelve dos `FunctionResponse`**, una con opciones de vuelos y otra con opciones de hoteles.

6.  **Turno 3: Modelo -> Respuesta al usuario**
    *   El modelo recibe las opciones de vuelos y hoteles. En lugar de reservar directamente, resume la información y pide confirmación.
    *   **Genera texto:** "He encontrado un vuelo con Iberia por 120€ y 3 hoteles céntricos disponibles desde 90€/noche: Hotel Palace, Hotel Ritz y Hotel Central. ¿Cuál quieres que reserve?"

7.  **Usuario:** "Reserva el vuelo de Iberia y el Hotel Central."

8.  **Turno 4: Modelo -> `reservar_vuelo` y `reservar_hotel`**
    *   Con la confirmación del usuario, el modelo procede a las llamadas finales.
    *   **Llama a las funciones:**
        *   `FunctionCall { Name = "reservar_vuelo", Args = { VueloId = "IBE123" } }`
        *   `FunctionCall { Name = "reservar_hotel", Args = { HotelId = "HC456" } }`

9.  **App -> Respuestas de las funciones**
    *   Se completan las reservas.
    *   **Devuelve dos `FunctionResponse`** con los números de confirmación.

10. **Turno 5: Modelo -> Respuesta Final**
    *   **Genera texto:** "¡Todo listo! Tu vuelo está reservado con el localizador `ABCDEF` y tu estancia en el Hotel Central tiene la confirmación `GHIJKL`. ¡Buen viaje!"

Este flujo avanzado demuestra cómo `Function Calling Multi-Turn` permite a una aplicación manejar tareas complejas, interactuar con el usuario para resolver ambigüedades y ejecutar una cadena de acciones lógicas para alcanzar un objetivo final.