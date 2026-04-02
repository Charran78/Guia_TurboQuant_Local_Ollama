# Guía General de Uso de TurboQuant: Compresión de Caché KV para LLMs

TurboQuant es una técnica de compresión del **caché KV** (Key-Value cache) desarrollada por Google Research que reduce drásticamente la memoria necesaria para mantener contextos largos en modelos de lenguaje grandes. Con TurboQuant, puedes:

- Multiplicar por 4-6 el tamaño de contexto sin aumentar el consumo de RAM.
- Ejecutar modelos de 7B o 13B en equipos con solo 8–16 GB de RAM.
- Mantener la calidad de respuesta incluso con cuantizaciones agresivas (3-4 bits).

Esta guía te enseñará cómo utilizar TurboQuant con **llama.cpp**, **Ollama** y **MLX** (Apple Silicon). Incluye ejemplos concretos, parámetros y consejos para hardware limitado.

---

## 1. ¿Qué es TurboQuant?

TurboQuant se centra en comprimir la **caché KV**, que almacena los estados intermedios de los tokens durante la generación. En lugar de reducir el tamaño del modelo (como la cuantización tradicional), TurboQuant:

- Usa técnicas de **precisión mixta y cuantización por canal** para comprimir la caché KV hasta 4,6 veces.
- Permite usar contextos enormes (ej. 32k tokens) en la misma memoria que antes ocupaba 4k.
- Es complementario a la cuantización del modelo (Q4_K_M, etc.).

**Ventajas clave:**  
✅ Más contexto en menos memoria  
✅ Mejor precisión en modelos muy comprimidos  
✅ Ideal para RAG, conversaciones largas o agentes.

---

## 2. Requisitos del sistema

- **CPU:** Procesador con soporte AVX2 (Intel Haswell o posterior, AMD Zen 2+).  
- **RAM:** 8 GB mínimo para modelos 3B, 16 GB para 7B con contexto largo.  
- **GPU (opcional):** NVIDIA con al menos 4 GB VRAM (para descargar algunas capas) o Apple Silicon (Metal).  
- **Sistema operativo:** Linux, Windows (WSL2 recomendado), macOS.

---

## 3. Opción A: Usar TurboQuant con llama.cpp (recomendado)

llama.cpp tiene soporte experimental de TurboQuant en sus versiones recientes. Es la forma más flexible y eficiente.

### 3.1 Obtener llama.cpp con soporte TurboQuant

#### Compilar desde código fuente
```bash
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp
# Asegúrate de tener la última versión que incluya TurboQuant
git checkout master   # o alguna rama específica como 'turbo' si existe
make -j$(nproc)       # o usa CMake: mkdir build && cd build && cmake .. && make -j
```

#### Alternativa: Descargar binarios precompilados
Algunos lanzamientos no oficiales incluyen soporte. Busca en el repositorio de releases de llama.cpp o en foros como r/LocalLLaMA.

### 3.2 Descargar un modelo cuantizado (GGUF)

Para aprovechar TurboQuant, usa modelos en formato GGUF con cuantización de 4 bits (Q4_K_M, Q4_K_S, etc.). Ejemplo con Qwen2.5-7B:

```bash
wget https://huggingface.co/bartowski/Qwen2.5-7B-Instruct-GGUF/resolve/main/Qwen2.5-7B-Instruct-Q4_K_M.gguf
```

### 3.3 Lanzar el servidor con TurboQuant

Activa la compresión de caché KV con los parámetros `--cache-type-k` y `--cache-type-v`:

```bash
./server -m Qwen2.5-7B-Instruct-Q4_K_M.gguf \
         --host 0.0.0.0 \
         --port 8080 \
         --ctx-size 16384 \
         --cache-type-k turbo3 \
         --cache-type-v turbo3 \
         --threads 4 \
         --n-gpu-layers 20   # si tienes GPU
```

- `--cache-type-k turbo3` y `--cache-type-v turbo3` indican el uso de TurboQuant (puede variar a `turbo` o `turbo8` según la versión).
- `--ctx-size` establece el tamaño de contexto máximo. Con TurboQuant, puedes duplicarlo o cuadriplicarlo sin problemas.
- `--threads` limita los hilos de CPU; deja uno libre para el sistema.

### 3.4 Probar la conexión

Usa curl para verificar:

```bash
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen",
    "messages": [{"role": "user", "content": "¿Qué es la compresión de caché KV?"}],
    "stream": true
  }'
```

Si ves la respuesta en streaming, todo funciona.

### 3.5 Integrar con tu aplicación

El servidor expone una API compatible con OpenAI, por lo que puedes conectar cualquier cliente que use `http://localhost:8080/v1` como base URL.

---

## 4. Opción B: Usar TurboQuant con Ollama

Ollama aún no integra TurboQuant de forma nativa en sus versiones estables, pero existen ramas experimentales y puedes crear modelos personalizados que usen kernels de TurboQuant si el binario subyacente lo soporta.

### 4.1 Obtener un modelo TurboQuant en formato Ollama

Busca en Hugging Face modelos GGUF que mencionen explícitamente `turboquant` o `turbo3`. También puedes descargar un modelo GGUF normal y crear un Modelfile con parámetros que activen la compresión.

### 4.2 Crear un Modelfile

Crea un archivo `Modelfile` con el siguiente contenido:

```dockerfile
# Ruta al archivo GGUF descargado
FROM ./qwen2.5-7b-turboquant.gguf

# Parámetros TurboQuant (dependen de la versión de Ollama/llama.cpp)
PARAMETER kv_cache_type q8_0   # o "turbo3", depende del soporte
PARAMETER num_ctx 32768        # contexto grande
PARAMETER num_thread 4          # ajusta según tu CPU

# Template para Qwen (ejemplo)
TEMPLATE """{{ if .System }}<|im_start|>system
{{ .System }}<|im_end|>
{{ end }}<|im_start|>user
{{ .Prompt }}<|im_end|>
<|im_start|>assistant
"""
PARAMETER stop "<|im_start|>"
PARAMETER stop "<|im_end|>"
```

### 4.3 Crear y ejecutar el modelo

```bash
ollama create qwen-turbo -f Modelfile
ollama run qwen-turbo
```

**Limitación:** La mayoría de las versiones oficiales de Ollama no incluyen los kernels TurboQuant. Si no ves mejora en el uso de memoria con contextos largos, considera compilar Ollama desde fuente con soporte TurboQuant o usar llama.cpp directamente.

---

## 5. Opción C: MLX (Apple Silicon)

Si tienes una Mac con chip M1/M2/M3, puedes usar MLX, que cuenta con implementaciones optimizadas de TurboQuant para la caché KV.

### 5.1 Instalar MLX

```bash
pip install mlx-lm
```

### 5.2 Cargar un modelo con TurboQuant

En Python, puedes activar la compresión de caché KV al cargar el modelo:

```python
from mlx_lm import load, generate

model, tokenizer = load("mlx-community/Qwen2.5-7B-Instruct-4bit", 
                        kv_cache_type="turbo3")  # Soporte experimental

response = generate(model, tokenizer, prompt="Hola", verbose=True)
```

Consulta la documentación de MLX para más detalles.

---

## 6. Verificar que TurboQuant está activo

### En llama.cpp
Los logs de inicio mostrarán líneas como:
```
KV cache type: turbo3
KV cache size: 512.00 MiB (for 8192 tokens)
```

### En Ollama
Revisa los logs del servidor (en Linux: `journalctl -u ollama -f`). Busca menciones a `kv_cache_type` o `turbo`.

### En MLX
Activa el logging para ver la configuración de la caché.

### Monitorización de memoria
- **Linux:** `htop` o `nvidia-smi` (si usas GPU).  
- **Windows:** Administrador de tareas > Rendimiento.  
- **Mac:** Monitor de actividad.

Si al aumentar `num_ctx` la RAM no se dispara, TurboQuant está funcionando.

---

## 7. Consejos para hardware limitado

- **Elige bien la cuantización del modelo:** Q4_K_M o Q3_K_S son las más equilibradas para CPU.
- **Limita los hilos:** `--threads 3` en un i5 de 4 núcleos evita saturar el sistema.
- **Reduce el tamaño de contexto inicial:** Empieza con 8192 y ve subiendo hasta que notes degradación.
- **Usa el modo WAL en SQLite:** Si tu app guarda conversaciones localmente, activa el journal WAL para evitar bloqueos.
- **Procesa en streaming y acumula escrituras:** No guardes cada token en disco; hazlo por lotes.

---

## 8. Solución de problemas comunes

| Problema | Posible causa | Solución |
|----------|---------------|----------|
| `error: unknown argument: --cache-type-k` | Versión de llama.cpp antigua | Actualiza a la última versión (master). |
| El modelo no responde o se cuelga | Falta de RAM o contexto demasiado grande | Reduce `--ctx-size` o usa un modelo más pequeño. |
| Respuesta muy lenta (menos de 1 token/s) | Demasiados hilos compitiendo | Reduce `--threads` a 2 o 3. |
| Ollama no reconoce `kv_cache_type` | Versión estable sin soporte | Usa llama.cpp o compila Ollama desde fuente. |
| La app se bloquea al recibir streaming | Escrituras excesivas en disco | Aplica buffer de tokens y guarda solo al final o cada X segundos. |

---

## 9. Conclusión

TurboQuant es una herramienta revolucionaria para ejecutar LLMs en hardware modesto, permitiendo contextos extensos sin aumentar la memoria. Con esta guía, puedes empezar a usarlo en tus proyectos, ya sea con llama.cpp (recomendado), Ollama o MLX.

**Recursos adicionales:**  
- [Repositorio de llama.cpp](https://github.com/ggerganov/llama.cpp)  
- [Paper de Google Research sobre TurboQuant](https://research.google/blog/turboquant-redefining-ai-efficiency-with-extreme-compression/)
- [Discusión en r/LocalLLaMA](https://www.reddit.com/r/LocalLLaMA/)  

Si tienes dudas específicas sobre tu hardware o modelo, no dudes en preguntar.
