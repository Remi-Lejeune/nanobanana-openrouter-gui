# 🍌 Nano Banana GUI

A simple and elegant GUI for AI image generation using OpenRouter's image models (Gemini 2.5 Flash Image & Gemini 3 Pro Image).

## Features

- **Image Generation** - Generate images from text prompts
- **Image-to-Image** - Upload an input image to guide generation
- **Model Selection** - Choose between Nano Banana (fast) and Nano Banana Pro (quality)
- **Temperature Control** - Adjust creativity from predictable to creative
- **History** - View and reuse previously generated images
- **Dark Mode** - Easy on the eyes, enabled by default
- **Image Actions** - Copy, download, or reuse generated images

## Setup

1. Clone the repository
2. Install dependencies:
   ```bash
   bun install
   ```
3. Start the dev server:
   ```bash
   bun run dev
   ```
4. Open http://localhost:5173 in your browser
5. Enter your [OpenRouter API key](https://openrouter.ai/keys)

## Usage

1. Enter your OpenRouter API key
2. (Optional) Upload an input image for image-to-image generation
3. Write a prompt describing the image you want
4. Select a model (Nano Banana or Nano Banana Pro)
5. Adjust temperature if desired
6. Click "Generate Image"

## Tech Stack

- Vue 3 (Composition API)
- Vite
- OpenRouter API

## License

MIT
