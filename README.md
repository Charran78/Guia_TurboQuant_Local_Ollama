# 🔥 TurboQuant – Guía de uso general para compresión de caché KV

Este repositorio contiene una guía práctica para usar **TurboQuant**, una técnica de compresión del caché KV desarrollada por Google Research, que permite ejecutar modelos de lenguaje grandes (LLMs) con contextos muy largos en hardware limitado.

Con TurboQuant puedes:
- Multiplicar por 4-6 el tamaño de contexto sin aumentar la memoria RAM.
- Ejecutar modelos de 7B o 13B en equipos con solo 8–16 GB de RAM.
- Mantener una alta calidad de respuesta incluso con cuantizaciones agresivas (3-4 bits).

## 📖 Contenido

- `turboquant_guide.md` – Guía completa con ejemplos para **llama.cpp**, **Ollama** y **MLX**.
- Parámetros recomendados para CPU antiguas (Intel i5 4ª gen, etc.).
- Consejos para optimizar el streaming y evitar bloqueos en aplicaciones cliente.
- Solución de problemas comunes.

## 🚀 ¿Cómo usar esta guía?

1. **Clona el repositorio** (o descarga el archivo markdown).
2. Abre `turboquant_guide.md` en tu editor o visor de Markdown favorito.
3. Sigue los pasos según tu hardware y el motor de inferencia que uses:
   - **Opción A (recomendada):** llama.cpp con soporte TurboQuant.
   - **Opción B:** Ollama (experimental).
   - **Opción C:** MLX para Apple Silicon.
4. Aplica los consejos de optimización para aplicaciones cliente (streaming, buffer de escritura, modo WAL en SQLite).

## 🧰 Requisitos mínimos

- **CPU:** Intel Haswell (2013+) o equivalente (soporte AVX2).
- **RAM:** 8 GB (modelos 3B) o 16 GB (modelos 7B).
- **SO:** Linux, Windows (WSL2), macOS.

> No se requiere GPU potente; TurboQuant está optimizado para CPU.

## 📝 Ejemplo rápido (llama.cpp)

```bash
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp
make -j4

# Descarga un modelo GGUF (ej. Qwen2.5-7B-Q4_K_M)
wget https://huggingface.co/bartowski/Qwen2.5-7B-Instruct-GGUF/resolve/main/Qwen2.5-7B-Instruct-Q4_K_M.gguf
```
# Lanza el servidor con TurboQuant
./server -m Qwen2.5-7B-Instruct-Q4_K_M.gguf \
         --ctx-size 16384 \
         --cache-type-k turbo3 \
         --cache-type-v turbo3 \
         --threads 4
🤝 Contribuciones
Si encuentras algún error o quieres añadir más ejemplos (por ejemplo, para otros modelos o hardware específico), eres bienvenido a abrir un issue o enviar un pull request.

📚 Enlaces de interés
Paper de Google Research sobre TurboQuant

Repositorio oficial de llama.cpp

Discusión en r/LocalLLaMA

📄 Licencia
Este repositorio es solo una guía de uso. El contenido se comparte con fines educativos. Los modelos y herramientas mencionados tienen sus propias licencias.

Hecho con ❤️ para la comunidad de IA local y hardware reciclado.
