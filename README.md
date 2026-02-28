# Kaksha-Audio-AMD
# Kaksha-Audio 🎧📚
> **AMD Slingshot Hackathon 2026 Submission** > *Track: AI in Education & Skilling*

![Kaksha Audio Banner](https://via.placeholder.com/1000x300?text=Kaksha-Audio:+Bridging+Literacy+with+Socratic+AI)

## 💡 The Problem
In India, millions of students struggle with reading comprehension due to language barriers and static, text-heavy textbooks. Standard Text-to-Speech (TTS) tools are robotic and monotonous, leading to low retention and disengagement.

## 🚀 The Solution
**Kaksha-Audio** is an AI-powered **"Socratic Audio Engine"** that transforms static textbook images into engaging, two-way audio podcasts.
Instead of just reading text, it uses Generative AI to simulate a realistic dialogue between a **Teacher** (who explains concepts) and a **Student** (who asks clarifying questions)—delivered in native Indian languages.

## ⚡ AMD Integration Strategy
Our architecture is designed to leverage **AMD Hardware** for both training-free inference and efficient scaling:
* **Edge Inference:** The Socratic Scripting Model (Llama 3 8B) is optimized to run on **AMD Ryzen™ AI (NPU)** enabled laptops, allowing for offline, low-latency lesson generation in rural schools.
* **Cloud Backend:** The core API orchestration handles concurrent requests using **AMD EPYC™ Processor-based** cloud instances (e.g., AWS T3a series), ensuring cost-effective scalability.
* **Open Ecosystem:** We utilize **ROCm™** optimized libraries to ensure our PyTorch and ONNX models run efficiently on AMD GPUs.

## 🌟 Key Features
* **📸 Snap-to-Learn:** Upload a photo of any textbook page (Science, History, Math).
* **🗣️ Socratic Dialogue:** automatically converts text into a "Teacher vs. Student" script.
* **🇮🇳 Multilingual Support:** Generates audio in Hindi, Tamil, Telugu, and "Hinglish."
* **💾 Offline First:** Optimized for low-bandwidth environments (2G/3G) with highly compressed audio.

## 🛠️ Tech Stack
* **Frontend:** React.js (Web), Flutter (Mobile)
* **Backend:** FastAPI (Python)
* **AI Engine:**
    * **Vision:** EasyOCR / Tesseract
    * **LLM:** Llama 3 (via Ollama/Groq)
    * **TTS:** Edge-TTS (Microsoft Edge Online Voices)
* **Database:** PostgreSQL (Supabase)

## 🗺️ Roadmap
- [ ] Implement core OCR pipeline.
- [ ] Integrate Llama 3 for script generation.
- [ ] Connect Edge-TTS for multi-speaker audio.
- [ ] Optimize inference for AMD Ryzen™ AI NPU (ONNX Runtime).

---
*Built with ❤️ by Team Sutra Coders*
