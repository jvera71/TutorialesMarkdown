Claro, aquí tienes un tutorial en español de España sobre la funcionalidad `Code Execution` de Google Gemini, explicando su propósito, interacciones avanzadas y mostrando el código de ejemplo que has proporcionado.

***

# Tutorial Avanzado: Ejecución de Código (Code Execution) con Google Gemini

## ¿Qué es y para qué sirve la Ejecución de Código en Gemini?

La **Ejecución de Código** (o `Code Execution`) es una de las "herramientas" (`Tool`) más potentes que puedes proporcionar a los modelos de Gemini. Imagina poder pedirle a un modelo de lenguaje que no solo te *diga* cómo resolver un problema matemático o analizar un conjunto de datos, sino que lo *resuelva* por ti, ejecutando código real para obtener una respuesta precisa.

En esencia, al habilitar `Code Execution`, le das a Gemini acceso a un **intérprete de código Python en un entorno seguro y aislado (sandbox)**. Cuando el modelo detecta que una pregunta puede resolverse de manera más eficiente o precisa mediante programación, puede:

1.  **Escribir código Python** de forma autónoma para abordar la tarea.
2.  **Ejecutar ese código** en su entorno aislado.
3.  **Utilizar el resultado** de esa ejecución para formular la respuesta final.

Esto es increíblemente útil para:
*   **Cálculos matemáticos complejos**: Resolver ecuaciones, realizar operaciones estadísticas, etc.
*   **Análisis de datos**: Procesar datos de ficheros (como CSV, JSON), extraer información y generar resúmenes.
*   **Manipulación de texto**: Ordenar listas, buscar patrones y otras tareas que son triviales con código pero complejas para un LLM puro.

A diferencia de simplemente generar fragmentos de código que tú tendrías que copiar y ejecutar, `Code Execution` cierra el círculo, permitiendo al modelo verificar sus propios razonamientos computacionales y devolverte un resultado ya validado.

### Ejemplo de Código Fuente (C#)

A continuación se muestra una función de prueba en C# que invoca al modelo Gemini con la herramienta `Code Execution` habilitada. No entraremos a explicar la sintaxis de las pruebas, sino el concepto que demuestran.

```csharp
[Fact]
// Ref: https://ai.google.dev/api/generate-content#code-execution
public async Task Generate_Content_Code_Execution()
{
    // Arrange
    var prompt = "What is the sum of the first 50 prime numbers?";
    var googleAi = new GoogleAI(_fixture.ApiKey);
    var model = _googleAi.GenerativeModel(model: _model,
        tools: [new Tool { CodeExecution = new() }]);

    // Act
    var response = await model.GenerateContent(prompt);

    // Assert
    response.Should().NotBeNull();
    response.Candidates.Should().NotBeNull().And.HaveCount(1);
    _output.WriteLine(string.Join(Environment.NewLine,
        response.Candidates[0].Content.Parts.Select(x =>
                x.Text)
            //                    .Where(t => !string.IsNullOrEmpty(t))
            .ToArray()));
}
```

En este caso, se le plantea al modelo una pregunta matemática: "¿Cuál es la suma de los 50 primeros números primos?". Un modelo de lenguaje tradicional podría intentar recordarlo o calcularlo mentalmente, con un alto riesgo de error. Sin embargo, al tener `Code Execution` activado, Gemini identifica que esta es una tarea computacional. Generará internamente un script de Python para encontrar los primeros 50 primos, sumarlos y, finalmente, devolverá el resultado numérico correcto como parte de su respuesta.

---

## Interacciones Avanzadas con Otras Funcionalidades de Gemini

El verdadero poder de `Code Execution` se desata cuando se combina con otras capacidades de Gemini. A continuación, exploramos escenarios avanzados.

### 1. Análisis de Datos a Gran Escala con la `File API` (`Generate_Content_Code_Execution_using_FileAPI`)

Esta es una de las combinaciones más prácticas y potentes. Permite al modelo analizar ficheros que le proporciones.

**Escenario avanzado:**
Imagina que tienes un fichero CSV (`ventas_anuales.csv`) con miles de registros de ventas y quieres obtener información compleja que requiere cálculos.

1.  **Subida del fichero**: Primero, utilizas la **File API** para subir tu fichero `ventas_anuales.csv` al almacenamiento de Google. Obtienes una referencia única (un `uri` o `name`) para ese fichero.
2.  **Prompt combinado**: Realizas una petición a Gemini combinando texto y la referencia al fichero.
    *   **Prompt**: "Analiza el fichero de ventas proporcionado. Calcula el total de ventas por categoría, la media de ingresos por transacción y genera un resumen de los tres productos más vendidos. Devuelve el resultado como una tabla en formato Markdown."
    *   **Herramientas**: Habilitas `Code Execution` y adjuntas el fichero usando su referencia de la `File API`.
3.  **Magia en el backend**:
    *   Gemini recibe el prompt y el acceso al fichero.
    *   Utiliza `Code Execution` para escribir un script de Python que probablemente use la librería `pandas` (disponible en su entorno).
    *   El script abrirá el CSV, agrupará por categoría, calculará las métricas solicitadas y ordenará los productos por ventas.
    *   Finalmente, usará los resultados obtenidos por el script para generar la tabla Markdown solicitada en la respuesta final.

**Resultado**: Obtienes un análisis de datos complejo y formateado sin escribir una sola línea de código de análisis tú mismo.

### 2. Combinando Lógica Interna y Externa con `Function Calling`

Es importante distinguir `Code Execution` de `Function Calling`:
*   **Function Calling**: El modelo te pide que ejecutes una función **de tu propio código** y le devuelvas el resultado. Es para interactuar con sistemas externos (tu base de datos, tu API, etc.).
*   **Code Execution**: El modelo ejecuta código Python **en su propio entorno aislado** para realizar cálculos o manipulaciones de datos.

**Escenario avanzado:**
Supongamos que has desarrollado una API interna para tu empresa que expone datos financieros en tiempo real, y quieres que Gemini use esos datos para hacer proyecciones.

1.  **Definición de herramientas**: Configuras Gemini con dos herramientas:
    *   Una `Function Declaration` para `get_financial_data(ticker, period)` que llama a tu API interna.
    *   `Code Execution` para el análisis.
2.  **Prompt complejo**: "Obtén los datos de cierre diarios de las acciones de 'GOOG' del último año y proyéctame su precio para el próximo mes usando una regresión polinómica de segundo grado."
3.  **Cadena de razonamiento (Chain-of-thought)**:
    *   **Paso 1 (Function Calling)**: Gemini detecta que necesita datos. Invoca a `get_financial_data('GOOG', '1y')`. Tu aplicación recibe esta llamada, consulta tu API y devuelve a Gemini un JSON con los datos históricos.
    *   **Paso 2 (Code Execution)**: Ahora que Gemini tiene los datos, ve que necesita realizar una regresión. Utiliza `Code Execution` para escribir un script con `scikit-learn` o `numpy` que procese el JSON recibido, ajuste un modelo de regresión polinómica y calcule los valores proyectados.
    *   **Paso 3 (Respuesta final)**: Con la proyección calculada, Gemini formula una respuesta en lenguaje natural: "Basado en los datos del último año, la proyección de precios para el próximo mes usando una regresión polinómica es..." y adjunta los valores numéricos.

### 3. Procesamiento de Información en Tiempo Real con `Grounding (Google Search)`

El `Grounding` permite a Gemini consultar Google Search para obtener información actualizada. Combinado con `Code Execution`, puede procesar y cuantificar esa información.

**Escenario avanzado:**
Quieres analizar el sentimiento del mercado sobre una criptomoneda basándote en noticias recientes.

1.  **Prompt**: "Busca las 10 noticias más recientes sobre Bitcoin. Analiza los titulares y calcula qué porcentaje de ellos expresan un sentimiento positivo, negativo o neutro."
2.  **Herramientas**: Habilitas `GoogleSearchRetrieval` (la herramienta de Grounding) y `Code Execution`.
3.  **Proceso**:
    *   **Paso 1 (Grounding)**: Gemini usa Google Search para encontrar los 10 titulares de noticias más recientes sobre Bitcoin.
    *   **Paso 2 (Code Execution)**: Con la lista de titulares, Gemini usa `Code Execution` para escribir un script de Python que:
        *   Itera sobre cada titular.
        *   Aplica una lógica simple de análisis de sentimiento (por ejemplo, buscando palabras clave como "sube", "récord", "cae", "crisis").
        *   Contabiliza los resultados.
        *   Calcula los porcentajes.
    *   **Paso 3 (Respuesta final)**: Gemini presenta un resumen: "De las 10 noticias más recientes, he encontrado que el 40% tienen un sentimiento positivo, el 50% negativo y el 10% neutro."

---

## Conclusión

La funcionalidad de **Ejecución de Código** transforma a Gemini de un generador de información a un **solucionador de problemas activo**. Al permitirle escribir y ejecutar código Python, desbloqueas la capacidad de realizar tareas analíticas y computacionales complejas de forma directa. Cuando se combina con otras herramientas como la `File API`, `Function Calling` o `Google Search`, se convierte en una pieza central para construir agentes de IA verdaderamente autónomos y potentes.