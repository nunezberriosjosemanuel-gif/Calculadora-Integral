# Calculadora de Integrales Definidas — Documentación del Proyecto

Aplicación de escritorio JavaFX que **resuelve integrales definidas paso a paso** (de forma simbólica cuando es posible, y numérica como respaldo), muestra el procedimiento en notación matemática y grafica la función junto con el área bajo la curva.

---

## 1. Stack tecnológico

| Componente | Detalle |
|---|---|
| Lenguaje | Java 17 |
| UI | JavaFX 17.0.2 (`javafx-controls`, `javafx-fxml`, `javafx-web`) |
| Motor de expresiones numéricas | [exp4j](https://www.objecthunter.net/exp4j/) 0.4.8 |
| Build | Maven (`javafx-maven-plugin`) |
| Clase principal | `com.calculadora.App` |
| Módulos (JPMS) | `module-info.java` declara y exporta `controllers`, `services`, `model.dto`, `model.entities`, `model.enums` |

La interfaz de cálculo y graficado no usa controles nativos de JavaFX para la matemática: está construida como **HTML/CSS/JS embebido dentro de un `WebView`**, generado como *string* Java y cargado con `engine.loadContent(...)`. La comunicación JS → Java se hace con un truco de puente: JavaScript llama a `window.alert("CMD|arg1|arg2|arg3")` y `MainController` intercepta esa alerta para disparar la lógica Java correspondiente (`CALCULAR`, `GRAFICAR`).

## 2. Estructura de paquetes

```
com.calculadora
├── App.java                      → punto de entrada (extends Application)
├── config/                       → wiring de dependencias, rutas de vistas
├── controllers/                  → controladores JavaFX (@FXML)
├── model/
│   ├── analyzer/                 → clasificación del tipo de función (Chain of Responsibility)
│   │   └── rules/                 → una regla detectora por tipo de función
│   ├── dto/                      → objetos de transferencia (Request/Response)
│   ├── entities/                 → datos de dominio (Integral, Result, Step, Function...)
│   ├── enums/                    → TipoFuncion, MetodoIntegracion, TipoGrafica
│   ├── graph/                    → generadores de datos de gráfica 2D/3D
│   ├── methods/                  → un método de integración simbólica por clase
│   ├── parser/                   → lexer/parser de expresiones (no conectado al flujo actual)
│   ├── repository/               → historial y ajustes
│   ├── simplifier/                → simplificación algebraica y factorización
│   ├── solver/                   → SolverEngine, orquestador central
│   └── strategies/               → interfaz Strategy + registro con prioridades
├── services/                     → capa de aplicación (validación, resolución, historial, export, pasos, gráfica)
└── utils/                        → formateo LaTeX, normalización de expresiones, utilidades matemáticas
```

## 3. Arquitectura: cómo se resuelve una integral

```
UI (WebView, JS)
   │  window.alert("CALCULAR|f(x)|a|b")
   ▼
MainController.handleAlert()
   │  normaliza texto → notación exp4j
   ▼
IntegralService.resolve(IntegralRequest)
   ├─► ValidationService   (valida sintaxis, dominio, división ambigua)
   ├─► SolverService       (delega en SolverEngine)
   │      └─► SolverEngine
   │             ├─► FunctionClassifier   → TipoFuncion (Chain of Responsibility)
   │             └─► StrategyRegistry     → candidatos ordenados por prioridad
   │                    └─► IntegrationStrategy.integrate()  (Strategy)
   │                           (si falla, prueba la siguiente; si todas fallan,
   │                            cae a integración numérica)
   └─► HistoryService      (guarda la consulta resuelta)
   ▼
IntegralResponse (antiderivada en LaTeX, resultado numérico, pasos)
   ▼
MainController construye el HTML del procedimiento y lo inyecta en el WebView
```

### Patrones de diseño usados deliberadamente

- **Strategy + Registry con prioridad** (`IntegrationStrategy` / `StrategyRegistry`): cada método de integración (sustitución, por partes, fracciones parciales, trigonométrica, etc.) decide por sí mismo si puede resolver una expresión (`canHandle`) y en qué orden intentarlo (`prioridad`). Agregar un método nuevo solo requiere crear la clase e inscribirla en `DependencyContainer`, sin tocar el resto (principio Abierto/Cerrado). Reemplaza un antiguo `StrategyFactory` con cadenas `if/else`.
- **Chain of Responsibility** (`FunctionClassifier` + `FunctionDetectorRule`): una lista ordenada de reglas (`TrigonometricDetectorRule`, `LogarithmicDetectorRule`, `ExponentialDetectorRule`, `IrrationalDetectorRule`, `RationalDetectorRule`, `QuadraticDetectorRule`, `LinearDetectorRule`, `PolynomialDetectorRule`, `ConstantDetectorRule`) clasifica el tipo de función.
- **Dependency Container manual** (`DependencyContainer` + `ServiceFactory`): no se usa un framework de inyección de dependencias; todo el cableado de servicios y estrategias vive en una sola clase estática, comentada explícitamente como el punto único donde registrar nuevas reglas/estrategias.
- **Orquestador único** (`SolverEngine`): reemplaza a un antiguo `AntiderivativeEngine` monolítico (600+ líneas mezclando parsing, reglas y formato) por una clase con una sola responsabilidad — coordinar clasificación + selección de estrategia.

## 4. Módulos de resolución simbólica (`model/methods`)

Cada clase implementa una técnica de integración de cálculo:

| Clase | Técnica |
|---|---|
| `PolynomialMethod` | Integración directa de polinomios |
| `ExponentialMethod` | Funciones exponenciales |
| `LogarithmicMethod` | Funciones logarítmicas |
| `TrigonometricMethod` | Funciones trigonométricas básicas |
| `TrigonometricSubstitutionMethod` | Sustitución trigonométrica |
| `SubstitutionMethod` | Sustitución simple (usa `AntiderivativeMethod` como base) |
| `IntegrationByPartsMethod` | Integración por partes |
| `PartialFractionsMethod` | Fracciones parciales (repetidas y no repetidas) |
| `RadicalMethod` | Funciones irracionales/radicales |
| `AntiderivativeMethod` / `AntiderivativeEngine` | Motor base de antiderivadas usado por varios métodos anteriores |

Si ninguna estrategia simbólica produce una antiderivada elemental, `SolverEngine` marca el resultado como `MetodoIntegracion.NUMERICA` y `SolverService` recurre a **cuadratura de Gauss-Legendre** (`AppConfig.NUMERIC_QUADRATURE_SUBINTERVALS = 2000` subintervalos) sobre la función completa.

## 5. Capa `services` (aplicación)

- **`ValidationService`** — valida que la función no esté vacía, detecta divisiones ambiguas sin paréntesis claros y verifica el dominio muestreando 400 puntos del intervalo.
- **`SolverService`** — punto de entrada al `SolverEngine`; arma el `IntegralResponse` final (antiderivada, valor definido, método usado).
- **`IntegralService`** — orquesta validación → resolución → guardado en historial para cada solicitud.
- **`HistoryService`** / **`HistoryRepository`** — guarda y lista el historial de integrales resueltas.
- **`GraphService`** — calcula el área bajo la curva reutilizando el *mismo* motor que el panel principal (se documenta explícitamente en el código que antes usaba una cuadratura independiente, lo que causaba inconsistencias entre el valor mostrado en el gráfico y el del procedimiento; ya corregido).
- **`StepService`** — convierte listas de términos en objetos `Step` numerados para mostrar el procedimiento.
- **`ExportService`** — exporta el resultado a texto plano.

## 6. Controladores (`controllers`)

- **`MainController`** (≈900 líneas) — el controlador real de la pantalla principal: genera el HTML/CSS/JS del "calculador" embebido, intercepta el puente JS↔Java, ejecuta `calcularIntegral()` y arma el HTML del procedimiento paso a paso (con fracciones y LaTeX vía `LatexGenerator`).
- **`GraphController`** — genera los puntos de la función y el área bajo la curva (muestreo de 300 puntos con margen automático) y construye el HTML/JS de la gráfica interactiva mostrada en `Graph.fxml`.
- **`IntegralController`** — controlador delgado que solo delega a `IntegralService`; no está conectado a ninguna vista FXML.
- **`HomeController`, `HistoryController`, `SettingsController`, `StepsController`, `HelpController`** — clases vacías (stubs), sin lógica ni referencias `@FXML`; existen como puntos de extensión pero no están activos en el flujo actual.

## 7. Vistas (`resources/com/calculadora`)

| Archivo | Descripción |
|---|---|
| `views/Main.fxml` | Ventana principal: un único `WebView` controlado por `MainController` |
| `views/Graph.fxml` | Ventana/panel de gráfica, controlado por `GraphController` |
| `style.css` | Hoja de estilos JavaFX (aplica sobre todo a los contenedores nativos, ya que el contenido matemático vive dentro del `WebView`) |

## 8. Utilidades (`utils`)

- **`LatexGenerator`** — convierte resultados numéricos a fracciones/LaTeX legibles para el paso a paso.
- **`ExpressionNormalizer`** — normaliza la expresión escrita por el usuario en dos variantes: una para evaluación numérica (`exp4j`) y otra para el análisis simbólico.
- **`MathUtils`** — evaluación numérica de expresiones ya resueltas.
- **`ExpressionFormatter`**, **`Validator`**, **`Constants`** — formateo y validaciones de apoyo.

## 9. Componentes presentes pero no conectados al flujo activo

El código incluye piezas que existen como paquete completo pero que ningún otro archivo referencia actualmente (probablemente restos de una iteración anterior o trabajo en progreso):

- `model/parser/` (`Lexer`, `Parser`, `FunctionParser`, `ExpressionTree`, `ExpressionValidator`) — un lexer/parser propio de expresiones que no es invocado desde `services` ni `controllers`.
- `model/graph/Graph3DGenerator.java` — generador de datos 3D, sin llamadas desde ningún controlador (la app solo grafica en 2D).
- `model/simplifier/AlgebraSimplifier.java` — no referenciado fuera de su propio archivo.
- Los controladores stub (`HomeController`, `HistoryController`, `SettingsController`, `StepsController`, `HelpController`) y `IntegralController`.

## 10. Cómo ejecutar

```bash
mvn clean javafx:run
```

(Requiere Maven y JDK 17+; JavaFX 17 y `exp4j` se resuelven automáticamente vía Maven.)

---

*Documento generado a partir del código fuente subido (`calculadoraintegral.zip`).*
