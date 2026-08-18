Arquitectura, Configuración e Implementación de un Sistema de Revisión Automática de Pull Requests en Docker con Agentes de IA de Terminal
El desarrollo de un sistema de revisión automática de código desplegado localmente permite mantener el control total sobre la infraestructura, reducir los costos de orquestación en la nube e integrar directamente herramientas de IA basadas en la línea de comandos (CLI Agents). Plataformas como Claude Code (en su modo headless), Antigravity CLI (agy), OpenCode u OpenClaw permiten ejecutar razonamiento agentico dentro de un contenedor Docker local1. Esto permite procesar diffs, ejecutar linters y publicar comentarios de revisión inline directamente en un repositorio privado de GitHub mediante una GitHub App1.
1. Arquitectura del Sistema Local Basado en Docker y CLI Agents
En este modelo, el backend corre de forma local o en un servidor interno dentro de un contenedor Docker. La arquitectura se compone de cinco módulos coordinados:
Punto de Entrada e Ingestión (Cloudflare Tunnel + Webhook Receiver):
Un túnel inverso (como cloudflared) expone de forma segura un puerto local hacia un dominio público con cifrado SSL. Cuando un desarrollador abre o actualiza una PR en un repositorio privado, GitHub envía un evento pull_request al endpoint expuesto por el túnel. Un servidor HTTP ligero en Python (FastAPI) o Node.js que corre dentro de Docker valida la firma criptográfica HMAC X-Hub-Signature-256 y encola la tarea.
Procesador de Trabajos y Espacio de Trabajo (Docker Local Sandbox):
El backend desencola la tarea y utiliza el CLI oficial de GitHub (gh) o git para clonar o descargar el diff unificado de la PR dentro de un directorio temporal montado en el contenedor.
Ejecución de Herramientas Estáticas (Linters):
Antes de llamar a la IA, el contenedor ejecuta herramientas deterministas de análisis estático (Ruff, ESLint, Bandit) sobre los archivos modificados. Las salidas de estas herramientas se recopilan en un archivo de contexto.
Invocación Programática del Agente de IA de Terminal: El backend evalúa el tipo de código o nivel de severidad del cambio y selecciona por código el modelo y agente adecuado (por ejemplo, invocando un agente especializado en ciberseguridad para backend o un agente rápido para frontend) mediante banderas de terminal1.
Agente de Publicación de Comentarios (GitHub REST API):
Un script Python/Node.js toma el archivo JSON generado por la herramienta CLI de terminal, verifica que las sugerencias apunten a números de línea válidos añadidos en el diff (RIGHT side) y envía una llamada HTTP POST autenticada mediante el token de la GitHub App hacia los endpoints oficiales de GitHub.
2. Selección y Programación de Agentes CLI de Terminal (Claude Code, Antigravity CLI, OpenCode y OpenClaw)
Todas estas herramientas CLI soportan la selección explícita mediante código de modelos, agentes y modos no interactivos (headless)1.
Claude Code
Claude Code soporta la bandera -p para ejecutar tareas desatendidas y retornar el resultado formateado a stdout:



Bash
docker exec -i ai-reviewer-container claude -p \
  "Revisa el diff adjunto en 'pr_diff.patch'. Genera un reporte de code review enfocado en bugs lógicos, seguridad (OWASP) y rendimiento. Devuelve el resultado en JSON estricto." \
  --output-format json \
  --dangerously-skip-permissions \
  --max-turns 5 \
  --max-budget-usd 0.50


--output-format json: Fuerza la salida estructurada para que la aplicación local pueda parsear los comentarios.
--dangerously-skip-permissions: Evita bloqueos por solicitudes de confirmación del usuario dentro del contenedor.
Antigravity CLI (agy)
Antigravity CLI de Google permite especificar tanto el modelo como el agente directamente a través de banderas de consola y archivos de definición1:
Selección de Modelo vía Flag (-m / --model): Permite elegir dinámicamente entre modelos como Gemini 3.1 Pro, Claude Sonnet o GPT-OSS1.
Definición de Agentes Personalizados: Se definen en la carpeta del proyecto en .agents/agents/{nombre_agente}/agent.md especificando las reglas y permisos del agente4.
Invocación de Agente por Código: Se invoca en modo no interactivo con -p pasando la instrucción del agente1.



Bash
# Invocación con modelo Gemini 3.1 Pro y agente de seguridad
docker exec -i ai-reviewer-container agy \
  -m gemini-3.1-pro \
  -p "/agent security-scanner 'Analiza los cambios en pr_diff.patch en busca de vulnerabilidades OWASP y genera JSON'"


OpenCode
OpenCode cuenta con soporte nativo para seleccionar modelos, agentes y subagentes desde la línea de comandos2:
Selección de Modelo (-m / --model): Recibe el formato proveedor/modelo (ej. anthropic/claude-3-5-sonnet, openai/gpt-4o, opencode/gpt-5 o modelos locales)2.
Selección de Agente (--agent): Permite pasar el nombre del agente a ejecutar (ej. --agent plan, --agent build o un agente personalizado creado con opencode agent create y guardado en .opencode/agents/)2.
Modo No Interactivo (-p / run / -f json): Ejecuta el prompt directamente sin TUI y retorna JSON2.



Bash
# Invocación no interactiva especificando agente 'review' y modelo 'claude-3-5-sonnet'
docker exec -i ai-reviewer-container opencode run \
  --agent review \
  --model anthropic/claude-3-5-sonnet \
  --format json \
  --prompt "Analiza el archivo pr_diff.patch y retorna el reporte en JSON"


OpenClaw
OpenClaw soporta la automatización de flujos de trabajo desatendidos mediante banderas de configuración en línea3:
Modo No Interactivo (--non-interactive): Desactiva las solicitudes interactivas por TTY3.
Selección de Modelo (--model): Asigna el modelo objetivo para la ejecución3.
Directorio de Agente (--agent-dir): Especifica la ruta donde reside la configuración del agente objetivo3.



Bash
docker exec -i ai-reviewer-container openclaw onboard \
  --non-interactive \
  --model claude-3-5-sonnet \
  --agent-dir /workspace/.openclaw/agents/security_reviewer


3. Roadmap de Prompts e Instrucciones para Agentes CLI
El backend en Python/Node.js decide cuál CLI invocar en función de los archivos del PR.
Fase 1: Extracción de Diff y Enrutamiento de Agente en el Backend



Python
# Ejemplo conceptual en Python dentro del backend Docker
def seleccionar_agente_y_comando(archivos_modificados):
    # Si hay cambios en archivos críticos de backend/seguridad
    if any(f.startswith("src/auth/") or f.endswith(".sql") for f in archivos_modificados):
        return {
            "cli": "opencode",
            "args": ["run", "--agent", "security-auditor", "--model", "anthropic/claude-3-5-sonnet", "-f", "json"]
        }
    # Para cambios estándar en frontend
    else:
        return {
            "cli": "agy",
            "args": ["-m", "gemini-3.1-pro", "-p"]
        }


Fase 2: Prompt de Invocación enviado al CLI Agent
Este prompt se envía dinámicamente como argumento al agente seleccionado1.
[PROMPT PARA EL CLI AGENT]
Estás ejecutando una revisión automática de código dentro de un contenedor aislado. Tu tarea es analizar el archivo 'pr_diff.patch' y las alertas de 'linter_output.txt' disponibles en el directorio actual.
[INSTRUCCIONES DE ANÁLISIS]
Identifica únicamente los errores críticos de seguridad (OWASP), problemas de lógica de negocio y fallos de rendimiento introducidos en el diff.
Descarta errores estéticos de formato si ya aparecen marcados en 'linter_output.txt'.
Para cada problema hallado, especifica la ruta exacta del archivo y el número de línea en la nueva versión del código (lado derecho del diff).
La respuesta DEBE ser únicamente un objeto JSON válido acorde al esquema indicado a continuación. No agregues saludos, ni explicaciones fuera del JSON.
[SCHEMA REQUERIDO]
{
"summary": "Resumen general del análisis",
"comments": [
{
"path": "ruta/al/archivo.py",
"line": 42,
"body": "🔴 Severidad: CRÍTICA\n\nDescripción del problema.\n\nsuggestion\ncódigo_corregido()\n"
}
]
}
4. Guía de Configuración Manual Paso a Paso en GitHub y Entorno Local
Para vincular su contenedor Docker local con el repositorio privado de GitHub sin exponer vulnerabilidades de red, siga este procedimiento paso a paso.
Paso 1: Configurar el Túnel Local (Cloudflare Tunnel)
En la máquina donde correrá Docker, instale cloudflared.
Inicie un túnel hacia el puerto de su aplicación local (por ejemplo, el puerto 8000):
Bash
cloudflared tunnel --url http://localhost:8000


Copie la URL pública HTTPS generada por Cloudflare (ejemplo: https://abcd-123-45-67.trycloudflare.com). Esta URL actuará como la dirección pública para los webhooks.
Paso 2: Registrar la GitHub App
Vaya a GitHub.com, haga clic en su foto de perfil en la esquina superior derecha y seleccione Settings (o Your organizations -> elija la organización -> Settings).
En el panel izquierdo, desplácese hasta el final y haga clic en Developer settings.
En el menú lateral, seleccione GitHub Apps y haga clic en New GitHub App.
En GitHub App name, ingrese un nombre único (ejemplo: local-docker-ai-reviewer).
En Homepage URL, ingrese la dirección de su repositorio privado.
En la sección Webhooks:
Asegúrese de que la casilla Active esté marcada.
En Webhook URL, pegue la URL generada por Cloudflare Tunnel seguida de la ruta /webhook (ejemplo: https://abcd-123-45-67.trycloudflare.com/webhook).
En Webhook secret, ingrese un secreto aleatorio y guárdelo de forma segura.
Paso 3: Asignar Permisos y Suscribir Eventos
En Repository permissions, configure:
Pull requests: Read & write (permite leer el diff y publicar comentarios).
Contents: Read-only (permite acceder a la estructura de archivos).
Metadata: Read-only (asignado por defecto).
En Subscribe to events, seleccione la casilla Pull request.
En Where can this GitHub App be installed?, seleccione Only on this account.
Haga clic en Create GitHub App.
Paso 4: Obtener Credenciales e Instalar
Copie el App ID mostrado en la página de resumen.
Desplácese a Private keys y haga clic en Generate a private key. Se descargará un archivo .pem. Mueva este archivo a la carpeta del proyecto local.
En el menú izquierdo, haga clic en Install App y seleccione Install al lado de su cuenta/organización.
Elija Only select repositories, seleccione el repositorio privado de interés y haga clic en Install.
Extraiga el Installation ID de la URL del navegador tras la instalación (el número final en .../installations/12345678).
5. Análisis de Viabilidad, Costos y Comparación de Ejecución Local
Ejecutar la herramienta principalmente en local dentro de Docker utilizando CLI agents cambia la estructura de costos frente a arquitecturas 100% cloud o plataformas SaaS comerciales como CodeRabbit.
Tabla Comparativa de Costos (Proyección 500 PRs / Mes)

Concepto / Componente
SaaS Comercial (CodeRabbit Pro)
Cloud Total (Serverless + APIs Frontend)
Local Docker + CLI Agent (Antigravity / OpenCode + API Key)
Local Docker + LLM 100% Local (OpenCode + Ollama)
Suscripción / Asientos
~$24 - $48 USD por dev/mes.
$0.00 USD.
$0.00 USD.
$0.00 USD.
Cómputo Backend
Incluido en la plataforma.
~$20.00 - $35.00 USD/mes (Cloud Run/Lambda).
$0.00 USD (Servidor local/PC de desarrollo).
$0.00 USD (Servidor local/PC de desarrollo).
Costo de API de IA
Incluido en la suscripción.
~$15.00 - $25.00 USD/mes.
Costo por token consumido (~$5.00 - $12.00 USD/mes).
$0.00 USD (Modelo corriendo en GPU local).
Inversión Hardware (CapEx)
$0.00 USD
$0.00 USD
$0.00 USD (Aprovecha equipos de desarrollo existentes).
~$2,000 - $3,500 USD (Si requiere servidor GPU dedicado).
Costo Operativo Mensual
~$240 - $480 USD/mes (Equipo 10 devs)
~$35.00 - $60.00 USD/mes
~$5.00 - $12.00 USD/mes
~$20.00 USD/mes (Consumo eléctrico).

Viabilidad Técnica y Ventajas de la Selección de Agentes vía CLI
Especialización Dinámica de Análisis: Poder seleccionar el agente y el modelo mediante banderas CLI (-m, --agent) permite derivar revisiones críticas de seguridad a modelos con razonamiento profundo (como Claude 3.5 Sonnet u OpenCode en modo security-auditor), mientras que cambios menores de interfaz se pueden evaluar con modelos ultrarrápidos y económicos (como Gemini 3.1 Pro o GPT-4o-mini)1.
Privacidad y Control Aumentados:
Al correr dentro de un contenedor Docker local, los procesos de clonación, ejecución de linters y análisis estático se realizan de forma aislada dentro de su red corporativa. Ningún archivo sin modificar abandona la infraestructura local.
Independencia de Tiempos de Límite (Timeouts):
Al ejecutar la lógica dentro de un contenedor local que procesa tareas en segundo plano mediante colas, no hay riesgo de sufrir interrupciones por timeouts HTTP en la recepción de webhooks de GitHub. El webhook responde 200 OK instantáneamente y el contenedor Docker ejecuta la revisión completa.
6. Conclusiones y Recomendaciones Estratégicas
Utilizar Banderas CLI para el Enrutamiento de Agentes: Configure el backend para que construya dinámicamente la llamada al CLI Agent. Para Antigravity CLI utilice -m <modelo> y especifique el agente en .agents/1. Para OpenCode utilice --model <proveedor/modelo> y --agent <nombre_agente>2. Para OpenClaw utilice --model y --agent-dir en modo --non-interactive3.
Garantizar la Salida JSON Estricta: Asegúrese de activar siempre las banderas de salida estructurada (--format json en OpenCode, --output-format json en Claude Code, o esquemas JSON requeridos en los prompts de Antigravity CLI) para que el publicador de la GitHub App pueda procesar las sugerencias sin fallos sintácticos1.
Fuentes citadas
Antigravity CLI: A Hands-On Guide to Google's Terminal Coding Agent - DEV Community, https://dev.to/arindam_1729/antigravity-cli-a-hands-on-guide-to-googles-terminal-coding-agent-5bc7
CLI | OpenCode, https://opencode.ai/docs/cli/
Onboarding (CLI) - OpenClaw Docs, https://docs.openclaw.ai/start/wizard
Agents Command (/agents) | Google Antigravity Docs, https://antigravity.google/docs/cli/commands/agents
Settings, Rendering & Keybindings | Google Antigravity Docs, https://antigravity.google/docs/cli/settings
Antigravity CLI Features, https://antigravity.google/docs/cli/features
Models | OpenCode, https://opencode.ai/docs/models/
Agents - OpenCode, https://opencode.ai/docs/agents/
OpenCode CLI: ejecuta prompts, agentes, sesiones y automatización, https://open-code.ai/es/docs/cli
opencode-ai/opencode: A powerful AI coding agent. Built for the terminal. · GitHub - GitHub, https://github.com/opencode-ai/opencode
