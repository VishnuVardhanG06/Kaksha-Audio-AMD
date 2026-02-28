# Design Document: Kaksha-Audio

## Executive Summary

Kaksha-Audio is an AI-powered Socratic Audio Engine that transforms static textbook images into engaging, interactive audio podcasts. The system addresses India's literacy gap by providing accessible education through natural dialogue between an AI Teacher and AI Student. The platform leverages AMD hardware optimization—AMD Ryzen™ AI NPUs for offline edge inference and AMD EPYC™ processors for scalable cloud deployment.

The architecture follows a microservices pattern with four primary layers: Client Layer (React.js/Flutter), API Gateway (FastAPI), Intelligence Layer (OCR, LLM, TTS microservices), and Data Layer (PostgreSQL/Supabase). The AI pipeline processes textbook images through EasyOCR for text extraction, Llama 3 for conversational script generation, and Edge-TTS for audio synthesis, supporting Hindi, Tamil, Telugu, and English.

Key technical innovations include offline-first design with model quantization for edge deployment, AMD NPU acceleration for inference workloads, and a dual-deployment strategy enabling both cloud-based scalability and edge-based accessibility in low-connectivity regions.

## Architecture

### System Overview

Kaksha-Audio employs a layered microservices architecture designed for both cloud and edge deployment:

```
┌─────────────────────────────────────────────────────────────┐
│                      Client Layer                            │
│  ┌──────────────────┐         ┌──────────────────┐         │
│  │   React.js Web   │         │  Flutter Mobile  │         │
│  │   Application    │         │   Application    │         │
│  └──────────────────┘         └──────────────────┘         │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                     API Gateway Layer                        │
│              FastAPI (AMD EPYC™ Optimized)                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   Auth       │  │   Routing    │  │ Rate Limiting│     │
│  │   Service    │  │   Service    │  │   Service    │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   Intelligence Layer                         │
│         (AMD Ryzen™ AI NPU / AMD EPYC™ Optimized)          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ OCR Service  │  │ Script Gen   │  │ TTS Service  │     │
│  │  (EasyOCR)   │  │  (Llama 3)   │  │  (Edge-TTS)  │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      Data Layer                              │
│              PostgreSQL (Supabase)                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   User DB    │  │  Content DB  │  │  Audio Store │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

### Deployment Strategies

**Cloud Deployment (High Connectivity Regions):**
- API Gateway and Intelligence Layer hosted on AMD EPYC™-based cloud instances (AWS T3a, Azure Dv5)
- Horizontal scaling with Kubernetes for handling 100-1000+ concurrent users
- Centralized PostgreSQL database on Supabase
- Full model weights for maximum accuracy

**Edge Deployment (Low Connectivity Regions):**
- Complete inference pipeline runs locally on AMD Ryzen™ AI enabled laptops
- Quantized models (4-bit/8-bit) for reduced memory footprint
- Local SQLite cache for offline content storage
- Periodic sync to cloud when connectivity available

### Technology Stack

**Frontend:**
- React.js 18+ with TypeScript for web application
- Flutter 3.0+ for cross-platform mobile (Android/iOS)
- State management: Redux (React), Provider (Flutter)
- HTTP client: Axios (React), Dio (Flutter)

**Backend:**
- Python 3.10+ with FastAPI framework
- Microservices: OCR Service, Script Generation Service, TTS Service
- Message queue: Redis for async task processing
- API documentation: OpenAPI/Swagger

**AI/ML Pipeline:**
- EasyOCR for multilingual text extraction
- Llama 3 (8B parameter model) via Ollama for script generation
- Edge-TTS for audio synthesis
- Model optimization: ONNX Runtime, ROCm for AMD GPU acceleration

**Infrastructure:**
- Docker containers for all services
- Kubernetes for orchestration (cloud deployment)
- PostgreSQL 14+ on Supabase for data persistence
- Object storage: Supabase Storage for audio files
- Monitoring: Prometheus + Grafana

## Components and Interfaces

### Client Layer Components

**Web Application (React.js):**
```typescript
interface ImageUploadComponent {
  acceptedFormats: ['image/png', 'image/jpeg', 'image/webp'];
  maxFileSize: 10485760; // 10MB in bytes
  
  methods: {
    validateImage(file: File): ValidationResult;
    uploadImage(file: File): Promise<UploadResponse>;
    displayPreview(file: File): void;
  }
}

interface AudioPlayerComponent {
  state: {
    isPlaying: boolean;
    currentTime: number;
    duration: number;
    currentSpeaker: 'teacher' | 'student';
  };
  
  methods: {
    play(): void;
    pause(): void;
    stop(): void;
    seek(timestamp: number): void;
    highlightSpeaker(speaker: string): void;
  }
}

interface HistoryComponent {
  methods: {
    fetchHistory(userId: string): Promise<ContentItem[]>;
    loadCachedAudio(contentId: string): Promise<AudioData>;
    deleteHistoryItem(contentId: string): Promise<void>;
  }
}
```

**Mobile Application (Flutter):**
```dart
class CameraService {
  Future<File> captureImage();
  Future<bool> validateImageQuality(File image);
  Future<File> compressImage(File image, int maxSizeBytes);
}

class OfflineSyncService {
  Future<void> queueProcessingRequest(ProcessingRequest request);
  Future<void> syncQueuedRequests();
  Future<List<ProcessingRequest>> getPendingQueue();
  Stream<SyncStatus> watchSyncStatus();
}

class LocalCacheService {
  Future<void> cacheAudioContent(String contentId, AudioData audio);
  Future<AudioData?> getCachedAudio(String contentId);
  Future<void> clearCache();
  Future<int> getCacheSize();
}
```

### API Gateway Layer

**FastAPI Application:**
```python
from fastapi import FastAPI, UploadFile, HTTPException, Depends
from fastapi.security import HTTPBearer
from pydantic import BaseModel

class ImageUploadRequest(BaseModel):
    language_preference: str  # 'hi', 'ta', 'te', 'en'
    grade_level: Optional[int]

class ProcessingStatusResponse(BaseModel):
    request_id: str
    status: str  # 'queued', 'processing', 'completed', 'failed'
    progress: int  # 0-100
    estimated_time_remaining: Optional[int]

class AudioResponse(BaseModel):
    audio_url: str
    duration: float
    script: List[DialogueLine]
    metadata: ContentMetadata

# API Endpoints
@app.post("/api/v1/upload")
async def upload_image(
    file: UploadFile,
    request: ImageUploadRequest,
    token: str = Depends(HTTPBearer())
) -> ProcessingStatusResponse:
    """Upload textbook image for processing"""
    pass

@app.get("/api/v1/status/{request_id}")
async def get_processing_status(
    request_id: str,
    token: str = Depends(HTTPBearer())
) -> ProcessingStatusResponse:
    """Check processing status"""
    pass

@app.get("/api/v1/audio/{content_id}")
async def get_audio(
    content_id: str,
    token: str = Depends(HTTPBearer())
) -> AudioResponse:
    """Retrieve generated audio content"""
    pass

@app.get("/api/v1/history")
async def get_user_history(
    user_id: str,
    limit: int = 20,
    offset: int = 0,
    token: str = Depends(HTTPBearer())
) -> List[ContentItem]:
    """Retrieve user's content history"""
    pass
```

**Rate Limiting and Authentication:**
```python
from fastapi_limiter import FastAPILimiter
from fastapi_limiter.depends import RateLimiter

# Rate limiting configuration
RATE_LIMITS = {
    "upload": "10/minute",  # 10 uploads per minute per user
    "status": "60/minute",  # 60 status checks per minute
    "audio": "30/minute"    # 30 audio retrievals per minute
}

# JWT authentication
class AuthService:
    def verify_token(self, token: str) -> UserContext:
        """Verify JWT token and return user context"""
        pass
    
    def create_token(self, user_id: str) -> str:
        """Create JWT token for authenticated user"""
        pass
```

### Intelligence Layer Components

**OCR Service (EasyOCR):**
```python
from easyocr import Reader
import numpy as np
from typing import List, Tuple

class OCRService:
    def __init__(self, languages: List[str], use_gpu: bool = True):
        """
        Initialize EasyOCR with specified languages
        languages: ['hi', 'ta', 'te', 'en']
        """
        self.reader = Reader(languages, gpu=use_gpu)
        self.min_confidence = 0.7
    
    def extract_text(self, image_path: str) -> OCRResult:
        """
        Extract text from image with layout preservation
        Returns: OCRResult with text, confidence, and bounding boxes
        """
        results = self.reader.readtext(image_path)
        
        # Filter by confidence threshold
        filtered_results = [
            (bbox, text, conf) 
            for bbox, text, conf in results 
            if conf >= self.min_confidence
        ]
        
        # Preserve layout by sorting by vertical position
        sorted_results = self._sort_by_layout(filtered_results)
        
        return OCRResult(
            text=self._combine_text(sorted_results),
            confidence=self._average_confidence(sorted_results),
            language=self._detect_language(sorted_results),
            layout=sorted_results
        )
    
    def _sort_by_layout(self, results: List[Tuple]) -> List[Tuple]:
        """Sort text blocks by vertical then horizontal position"""
        return sorted(results, key=lambda x: (x[0][0][1], x[0][0][0]))
    
    def optimize_for_amd_npu(self):
        """Configure for AMD Ryzen AI NPU acceleration"""
        # Use ONNX Runtime with AMD execution provider
        pass

class OCRResult:
    text: str
    confidence: float
    language: str
    layout: List[TextBlock]
```

**Script Generation Service (Llama 3):**
```python
from ollama import Client
import json
from typing import List

class ScriptGenerationService:
    def __init__(self, model_name: str = "llama3:8b"):
        self.client = Client()
        self.model = model_name
        self.system_prompt = """You are an expert educational content creator. 
        Generate a Socratic dialogue between a Teacher and a Student based on the 
        provided textbook content. The Teacher explains concepts clearly, and the 
        Student asks thoughtful clarifying questions. Output must be valid JSON."""
    
    def generate_script(
        self, 
        extracted_text: str, 
        language: str,
        grade_level: int = 8
    ) -> DialogueScript:
        """
        Generate conversational script from extracted text
        Returns: DialogueScript with speaker-labeled dialogue
        """
        prompt = self._build_prompt(extracted_text, language, grade_level)
        
        response = self.client.generate(
            model=self.model,
            prompt=prompt,
            system=self.system_prompt,
            format="json"
        )
        
        script_data = json.loads(response['response'])
        return self._parse_script(script_data, language)
    
    def _build_prompt(
        self, 
        text: str, 
        language: str, 
        grade_level: int
    ) -> str:
        """Build prompt for Llama 3 with context"""
        return f"""
        Content: {text}
        Language: {language}
        Grade Level: {grade_level}
        
        Create a dialogue with:
        - Teacher explaining the concept
        - Student asking at least 3 clarifying questions
        - Natural conversation flow
        - Language appropriate for grade {grade_level}
        
        Output JSON format:
        {{
          "dialogue": [
            {{"speaker": "teacher", "text": "...", "timestamp": 0.0}},
            {{"speaker": "student", "text": "...", "timestamp": 5.2}}
          ],
          "topic": "...",
          "key_concepts": [...]
        }}
        """
    
    def _parse_script(self, script_data: dict, language: str) -> DialogueScript:
        """Parse and validate generated script"""
        dialogue_lines = []
        for line in script_data['dialogue']:
            dialogue_lines.append(DialogueLine(
                speaker=line['speaker'],
                text=line['text'],
                timestamp=line.get('timestamp', 0.0),
                language=language
            ))
        
        # Validate minimum student questions
        student_lines = [l for l in dialogue_lines if l.speaker == 'student']
        if len(student_lines) < 3:
            raise ValueError("Script must contain at least 3 student questions")
        
        return DialogueScript(
            dialogue=dialogue_lines,
            topic=script_data['topic'],
            key_concepts=script_data['key_concepts'],
            language=language
        )

class DialogueScript:
    dialogue: List[DialogueLine]
    topic: str
    key_concepts: List[str]
    language: str

class DialogueLine:
    speaker: str  # 'teacher' or 'student'
    text: str
    timestamp: float
    language: str
```

**TTS Service (Edge-TTS):**
```python
import edge_tts
import asyncio
from pydub import AudioSegment
from typing import List

class TTSService:
    def __init__(self):
        self.voice_mapping = {
            'hi': {
                'teacher': 'hi-IN-MadhurNeural',
                'student': 'hi-IN-SwaraNeural'
            },
            'ta': {
                'teacher': 'ta-IN-ValluvarNeural',
                'student': 'ta-IN-PallaviNeural'
            },
            'te': {
                'teacher': 'te-IN-MohanNeural',
                'student': 'te-IN-ShrutiNeural'
            },
            'en': {
                'teacher': 'en-IN-NeerjaNeural',
                'student': 'en-IN-PrabhatNeural'
            }
        }
        self.pause_duration = 500  # milliseconds between speakers
    
    async def synthesize_audio(
        self, 
        script: DialogueScript
    ) -> AudioFile:
        """
        Synthesize audio from dialogue script
        Returns: AudioFile with MP3 data and metadata
        """
        audio_segments = []
        
        for line in script.dialogue:
            # Get appropriate voice for speaker and language
            voice = self.voice_mapping[script.language][line.speaker]
            
            # Generate audio for this line
            communicate = edge_tts.Communicate(line.text, voice)
            audio_data = b""
            
            async for chunk in communicate.stream():
                if chunk["type"] == "audio":
                    audio_data += chunk["data"]
            
            # Convert to AudioSegment
            segment = AudioSegment.from_mp3(io.BytesIO(audio_data))
            audio_segments.append(segment)
            
            # Add pause between speakers
            pause = AudioSegment.silent(duration=self.pause_duration)
            audio_segments.append(pause)
        
        # Combine all segments
        final_audio = sum(audio_segments)
        
        # Export as MP3 with 128kbps bitrate
        output_buffer = io.BytesIO()
        final_audio.export(
            output_buffer, 
            format="mp3", 
            bitrate="128k"
        )
        
        return AudioFile(
            data=output_buffer.getvalue(),
            duration=len(final_audio) / 1000.0,  # Convert to seconds
            format="mp3",
            bitrate=128000
        )

class AudioFile:
    data: bytes
    duration: float
    format: str
    bitrate: int
```

## Data Models

### Database Schema (PostgreSQL)

```sql
-- Users table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    language_preference VARCHAR(10) DEFAULT 'en',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Content table
CREATE TABLE content (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    source_image_url TEXT,
    extracted_text TEXT NOT NULL,
    language VARCHAR(10) NOT NULL,
    grade_level INTEGER,
    topic VARCHAR(255),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Scripts table
CREATE TABLE scripts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content_id UUID REFERENCES content(id) ON DELETE CASCADE,
    dialogue_json JSONB NOT NULL,
    key_concepts TEXT[],
    created_at TIMESTAMP DEFAULT NOW()
);

-- Audio files table
CREATE TABLE audio_files (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    script_id UUID REFERENCES scripts(id) ON DELETE CASCADE,
    file_url TEXT NOT NULL,
    duration FLOAT NOT NULL,
    format VARCHAR(10) DEFAULT 'mp3',
    bitrate INTEGER DEFAULT 128000,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Processing queue table (for async processing)
CREATE TABLE processing_queue (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    content_id UUID REFERENCES content(id) ON DELETE CASCADE,
    status VARCHAR(20) DEFAULT 'queued',
    progress INTEGER DEFAULT 0,
    error_message TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Indexes for performance
CREATE INDEX idx_content_user_id ON content(user_id);
CREATE INDEX idx_scripts_content_id ON scripts(content_id);
CREATE INDEX idx_audio_files_script_id ON audio_files(script_id);
CREATE INDEX idx_processing_queue_status ON processing_queue(status);
CREATE INDEX idx_processing_queue_user_id ON processing_queue(user_id);

-- Row-level security policies
ALTER TABLE content ENABLE ROW LEVEL SECURITY;
CREATE POLICY content_user_policy ON content
    FOR ALL USING (auth.uid() = user_id);

ALTER TABLE scripts ENABLE ROW LEVEL SECURITY;
CREATE POLICY scripts_user_policy ON scripts
    FOR ALL USING (
        content_id IN (
            SELECT id FROM content WHERE user_id = auth.uid()
        )
    );
```

### Data Transfer Objects

```python
from pydantic import BaseModel, Field
from typing import List, Optional
from datetime import datetime
from uuid import UUID

class UserModel(BaseModel):
    id: UUID
    email: str
    language_preference: str = 'en'
    created_at: datetime

class ContentModel(BaseModel):
    id: UUID
    user_id: UUID
    source_image_url: Optional[str]
    extracted_text: str
    language: str
    grade_level: Optional[int]
    topic: Optional[str]
    created_at: datetime

class DialogueLineModel(BaseModel):
    speaker: str = Field(..., pattern='^(teacher|student)$')
    text: str
    timestamp: float = Field(..., ge=0.0)

class ScriptModel(BaseModel):
    id: UUID
    content_id: UUID
    dialogue: List[DialogueLineModel]
    key_concepts: List[str]
    created_at: datetime

class AudioFileModel(BaseModel):
    id: UUID
    script_id: UUID
    file_url: str
    duration: float = Field(..., gt=0.0)
    format: str = 'mp3'
    bitrate: int = 128000
    created_at: datetime

class ProcessingQueueModel(BaseModel):
    id: UUID
    user_id: UUID
    content_id: UUID
    status: str = Field(..., pattern='^(queued|processing|completed|failed)$')
    progress: int = Field(..., ge=0, le=100)
    error_message: Optional[str]
    created_at: datetime
    updated_at: datetime
```

## Data Flow

### End-to-End Processing Pipeline

```
1. Image Upload
   ├─> Client validates image (format, size)
   ├─> Client uploads to API Gateway
   └─> API Gateway stores in object storage
       └─> Returns request_id

2. OCR Processing
   ├─> API Gateway queues OCR task
   ├─> OCR Service downloads image
   ├─> EasyOCR extracts text (AMD NPU accelerated)
   ├─> Text stored in database
   └─> Updates processing status (33% complete)

3. Script Generation
   ├─> Script Service retrieves extracted text
   ├─> Llama 3 generates dialogue (AMD NPU/GPU accelerated)
   ├─> Validates minimum 3 student questions
   ├─> Stores script JSON in database
   └─> Updates processing status (66% complete)

4. Audio Synthesis
   ├─> TTS Service retrieves script
   ├─> Edge-TTS synthesizes each dialogue line
   ├─> Combines audio with pauses
   ├─> Uploads MP3 to object storage
   ├─> Stores audio metadata in database
   └─> Updates processing status (100% complete)

5. Client Retrieval
   ├─> Client polls status endpoint
   ├─> When complete, fetches audio URL
   ├─> Downloads and caches audio
   └─> Displays player interface
```

### Offline Mode Data Flow

```
1. Local Processing (No Internet)
   ├─> User captures image
   ├─> Local OCR Service processes (quantized EasyOCR)
   ├─> Local Script Service generates dialogue (quantized Llama 3)
   ├─> Local TTS Service synthesizes audio
   ├─> Stores in local SQLite cache
   └─> Marks for sync when online

2. Sync Process (Internet Restored)
   ├─> Background service detects connectivity
   ├─> Uploads cached content to cloud
   ├─> Syncs with user's cloud history
   └─> Enables cross-device access
```

## Hardware Optimization

### AMD Ryzen™ AI NPU Integration

**NPU Acceleration for Edge Inference:**

The AMD Ryzen™ AI processors feature dedicated Neural Processing Units (NPUs) that provide up to 16 TOPS (Tera Operations Per Second) for AI workloads. Kaksha-Audio leverages these NPUs for offline inference:

```python
import onnxruntime as ort

class AMDNPUOptimizer:
    def __init__(self):
        # Configure ONNX Runtime for AMD NPU
        self.session_options = ort.SessionOptions()
        self.session_options.graph_optimization_level = (
            ort.GraphOptimizationLevel.ORT_ENABLE_ALL
        )
        
        # Use AMD execution provider
        self.providers = [
            ('MIGraphXExecutionProvider', {
                'device_id': 0,
                'arena_extend_strategy': 'kSameAsRequested',
            }),
            'CPUExecutionProvider'
        ]
    
    def optimize_ocr_model(self, model_path: str):
        """Optimize EasyOCR for AMD NPU"""
        session = ort.InferenceSession(
            model_path,
            sess_options=self.session_options,
            providers=self.providers
        )
        return session
    
    def optimize_llm_model(self, model_path: str):
        """Optimize Llama 3 for AMD NPU with quantization"""
        # Use 4-bit quantization for edge deployment
        session = ort.InferenceSession(
            model_path,
            sess_options=self.session_options,
            providers=self.providers
        )
        return session
```

**Model Quantization for Edge Deployment:**

```python
from optimum.onnxruntime import ORTQuantizer
from optimum.onnxruntime.configuration import AutoQuantizationConfig

class ModelQuantizer:
    @staticmethod
    def quantize_for_edge(model_path: str, output_path: str):
        """
        Quantize models to 4-bit/8-bit for AMD Ryzen AI deployment
        Reduces memory footprint while maintaining accuracy
        """
        quantizer = ORTQuantizer.from_pretrained(model_path)
        
        # Configure dynamic quantization
        qconfig = AutoQuantizationConfig.avx512_vnni(
            is_static=False,
            per_channel=True
        )
        
        quantizer.quantize(
            save_dir=output_path,
            quantization_config=qconfig
        )
```

**Performance Targets:**
- OCR Processing: 3-5 seconds on AMD Ryzen AI (vs 8-10 seconds on CPU)
- Script Generation: 10-15 seconds on AMD Ryzen AI (vs 30-40 seconds on CPU)
- Total Pipeline: <60 seconds on edge devices

### AMD EPYC™ Cloud Optimization

**Multi-Core Parallelism:**

AMD EPYC™ processors provide up to 96 cores per socket, enabling massive parallelism for concurrent request handling:

```python
from fastapi import FastAPI
import uvicorn
from concurrent.futures import ProcessPoolExecutor

class EPYCOptimizedAPI:
    def __init__(self, num_workers: int = 32):
        """
        Configure FastAPI for AMD EPYC multi-core utilization
        num_workers: Number of worker processes (typically cores - 4)
        """
        self.app = FastAPI()
        self.executor = ProcessPoolExecutor(max_workers=num_workers)
    
    def run(self):
        """Run with optimized worker configuration"""
        uvicorn.run(
            self.app,
            host="0.0.0.0",
            port=8000,
            workers=32,  # Leverage EPYC cores
            loop="uvloop",  # High-performance event loop
            limit_concurrency=1000
        )
```

**ROCm GPU Acceleration:**

For cloud deployments with AMD GPUs, leverage ROCm for accelerated inference:

```python
import torch

class ROCmAccelerator:
    def __init__(self):
        if torch.cuda.is_available():
            self.device = torch.device("cuda")
            # ROCm is compatible with CUDA API
            print(f"Using AMD GPU: {torch.cuda.get_device_name(0)}")
        else:
            self.device = torch.device("cpu")
    
    def load_model_on_gpu(self, model):
        """Load model on AMD GPU with ROCm"""
        return model.to(self.device)
```

**Deployment Specifications:**

Cloud Infrastructure (AMD EPYC™):
- Instance Type: AWS T3a.xlarge or Azure Dv5 series
- vCPUs: 4-8 AMD EPYC cores
- Memory: 16-32 GB RAM
- Storage: 100 GB SSD for model weights
- Expected Throughput: 100-500 concurrent users per instance

Edge Infrastructure (AMD Ryzen™ AI):
- Processor: AMD Ryzen 7 7840U or higher (with NPU)
- Memory: 16 GB RAM minimum
- Storage: 50 GB for quantized models and cache
- NPU: 16+ TOPS for AI acceleration
- Expected Performance: 30-60 second processing time


## Correctness Properties

A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

### Property 1: Image Upload Validation

*For any* image file uploaded by a user, the system should validate that the file format is one of PNG, JPEG, or WEBP, the file size is at most 10MB, and the image quality meets minimum standards before accepting the upload.

**Validates: Requirements 1.2, 1.3**

### Property 2: OCR Multilingual Support

*For any* image containing text in Hindi, Tamil, Telugu, or English, the OCR Engine should successfully extract the text and correctly identify the source language.

**Validates: Requirements 2.1, 5.3**

### Property 3: OCR Confidence Threshold

*For any* successful text extraction, the OCR Engine should return results with confidence scores of at least 0.7 for all extracted text segments.

**Validates: Requirements 2.2**

### Property 4: OCR Layout Preservation

*For any* image with structured text (paragraphs, lists, etc.), the OCR Engine should preserve the original layout and paragraph structure in the extracted text output.

**Validates: Requirements 2.4**

### Property 5: Script Structure Completeness

*For any* generated dialogue script, the output should be valid JSON containing speaker labels (teacher/student), text content, timing metadata, at least 3 student questions, and maintain consistent language throughout the dialogue.

**Validates: Requirements 3.1, 3.3, 3.4, 3.6, 5.4**

### Property 6: TTS Audio Format Compliance

*For any* synthesized audio file, the output should be in MP3 format with a bitrate of at least 128kbps, contain distinct voices for teacher and student speakers, and include pauses of at least 0.5 seconds between speaker turns.

**Validates: Requirements 4.1, 4.3, 4.4**

### Property 7: TTS Multilingual Voice Support

*For any* script in Hindi, Tamil, Telugu, or English, the TTS Engine should synthesize audio using appropriate native speaker voice models for the specified language.

**Validates: Requirements 4.2, 5.5**

### Property 8: TTS Metadata Completeness

*For any* completed audio synthesis, the system should return both the audio file URL and accurate total duration metadata.

**Validates: Requirements 4.5**

### Property 9: Language Preference Persistence

*For any* user language preference selection, the Client Layer should persist the selection such that it is retained across application restarts and future sessions.

**Validates: Requirements 5.2**

### Property 10: Offline Content Caching

*For any* audio content generated while online, the Client Layer should cache the content locally such that it remains accessible for playback when the device is offline.

**Validates: Requirements 6.2**

### Property 11: Offline Request Queuing

*For any* image processing request initiated while offline, the system should queue the request and automatically sync it to the cloud when internet connectivity is restored.

**Validates: Requirements 6.3**

### Property 12: Audio Playback Progress Tracking

*For any* audio file being played, the Client Layer should display accurate current timestamp and total duration, and support seeking to any valid position within the audio.

**Validates: Requirements 7.2, 7.3**

### Property 13: Speaker Highlighting Synchronization

*For any* audio playback in progress, the Client Layer should highlight the current speaker (teacher or student) in sync with the audio timeline.

**Validates: Requirements 7.4**

### Property 14: Content Storage Completeness

*For any* completed audio generation, the Data Layer should store the audio file, the complete script JSON, source image metadata, and associate all items with the user's account for cross-device access.

**Validates: Requirements 8.1, 8.4**

### Property 15: History Item Retrieval Without Reprocessing

*For any* previously processed content item selected from history, the Client Layer should load the cached audio without triggering reprocessing of the original image.

**Validates: Requirements 8.3**

### Property 16: Complete Content Deletion

*For any* history item deleted by a user, the system should remove all associated files (audio, script, image) and metadata from both local cache and cloud storage.

**Validates: Requirements 8.5**

### Property 17: Hardware Performance Metrics Logging

*For any* inference operation (OCR, script generation, TTS), the Intelligence Layer should monitor and log hardware utilization metrics including NPU/GPU usage, processing time, and memory consumption.

**Validates: Requirements 10.4**

### Property 18: API Authentication and Rate Limiting

*For any* incoming API request, the API Gateway should validate the authentication token and enforce rate limits according to the endpoint-specific configuration (10/min for uploads, 60/min for status checks, 30/min for audio retrieval).

**Validates: Requirements 11.2**

### Property 19: API Request Routing

*For any* validated API request, the API Gateway should correctly route the request to the appropriate microservice in the Intelligence Layer based on the endpoint and request type.

**Validates: Requirements 11.3**

### Property 20: Standardized Error Responses

*For any* error condition encountered by the API Gateway, the system should return a standardized error response containing the appropriate HTTP status code and a descriptive error message.

**Validates: Requirements 11.4**

### Property 21: Password Encryption Strength

*For any* new user account creation, the system should encrypt the password using bcrypt with a minimum of 12 rounds before storing in the database.

**Validates: Requirements 12.1**

### Property 22: TLS Encryption for Data Transmission

*For any* data transmission between client and server, the API Gateway should enforce HTTPS with TLS 1.3 or higher.

**Validates: Requirements 12.2**

### Property 23: Offline Content Encryption

*For any* content stored locally in offline mode, the Client Layer should encrypt the data using AES-256 before writing to local storage.

**Validates: Requirements 12.5**

### Property 24: Error Retry Logic

*For any* Script Generator error, the system should automatically retry the operation up to 3 times before returning a failure status to the user.

**Validates: Requirements 15.2**

### Property 25: Component Failure Logging

*For any* component failure in the Intelligence Layer, the system should log detailed error information including timestamp, component name, error type, and stack trace for debugging purposes.

**Validates: Requirements 15.4**

## Error Handling

### Error Categories and Strategies

**Input Validation Errors:**
- Invalid image format or size: Return 400 Bad Request with specific validation failure details
- Missing authentication token: Return 401 Unauthorized
- Malformed request body: Return 400 Bad Request with JSON schema validation errors

**Processing Errors:**
- OCR extraction failure: Suggest image quality improvements (better lighting, higher resolution, clearer text)
- Script generation failure: Retry up to 3 times with exponential backoff (1s, 2s, 4s)
- TTS synthesis failure: Queue request for delayed processing and notify user via status endpoint

**Infrastructure Errors:**
- Database connection loss: Return 503 Service Unavailable with Retry-After header
- Object storage unavailable: Queue uploads and retry with exponential backoff
- Microservice timeout: Return 504 Gateway Timeout after 30 seconds

**Resource Errors:**
- Rate limit exceeded: Return 429 Too Many Requests with Retry-After header
- Storage quota exceeded: Return 507 Insufficient Storage with upgrade prompt
- Concurrent user limit reached: Queue request or return 503 with estimated wait time

### Error Response Format

All error responses follow a standardized JSON structure:

```json
{
  "error": {
    "code": "OCR_EXTRACTION_FAILED",
    "message": "Failed to extract text from image",
    "details": "Image quality too low. Please ensure good lighting and clear text.",
    "timestamp": "2024-01-15T10:30:00Z",
    "request_id": "req_abc123",
    "suggestions": [
      "Use better lighting",
      "Increase image resolution",
      "Ensure text is in focus"
    ]
  }
}
```

### Graceful Degradation

**Offline Mode Fallback:**
- When cloud services unavailable, automatically switch to local inference pipeline
- Use quantized models with slightly reduced accuracy but maintained functionality
- Queue cloud-dependent operations for later sync

**Partial Processing:**
- If OCR succeeds but script generation fails, save extracted text for manual review
- If script generation succeeds but TTS fails, allow user to read transcript while TTS retries
- Store intermediate results to avoid complete reprocessing on retry

**User Notification:**
- Real-time status updates via WebSocket or polling
- Clear error messages with actionable suggestions
- Progress indicators showing which pipeline stage failed

## Testing Strategy

### Dual Testing Approach

Kaksha-Audio employs both unit testing and property-based testing to ensure comprehensive coverage:

**Unit Tests:** Focus on specific examples, edge cases, and integration points
**Property Tests:** Verify universal properties across randomized inputs (minimum 100 iterations per test)

### Property-Based Testing Configuration

**Framework Selection:**
- Python backend: Hypothesis library for property-based testing
- TypeScript frontend: fast-check library for property-based testing
- Flutter mobile: Built-in test framework with custom property test utilities

**Test Configuration:**
```python
# Python example using Hypothesis
from hypothesis import given, settings, strategies as st

@settings(max_examples=100)
@given(
    image_file=st.binary(min_size=1024, max_size=10*1024*1024),
    file_format=st.sampled_from(['png', 'jpeg', 'webp'])
)
def test_image_upload_validation(image_file, file_format):
    """
    Feature: kaksha-audio, Property 1: Image Upload Validation
    For any image file, validate format and size constraints
    """
    result = validate_image(image_file, file_format)
    assert result.is_valid == (len(image_file) <= 10*1024*1024)
    assert result.format in ['png', 'jpeg', 'webp']
```

**Property Test Tags:**
Each property test must include a comment tag referencing the design document:
```python
# Feature: kaksha-audio, Property 5: Script Structure Completeness
```

### Unit Testing Strategy

**Component-Level Tests:**
- OCR Service: Test with sample images in each supported language
- Script Generator: Test with various text lengths and complexity levels
- TTS Service: Test with different dialogue structures and languages
- API Gateway: Test authentication, rate limiting, and routing logic

**Integration Tests:**
- End-to-end pipeline: Image upload → OCR → Script → TTS → Storage
- Offline sync: Queue requests offline, verify sync when online
- Cross-device access: Create content on one device, access from another

**Edge Cases and Error Conditions:**
- Empty images (no text)
- Corrupted image files
- Extremely long text (>10,000 words)
- Mixed language content
- Network interruptions during processing
- Concurrent requests from same user
- Storage quota exhaustion

### Performance Testing

**Load Testing:**
- Simulate 100-1000 concurrent users on AMD EPYC cloud instances
- Measure response times under various load conditions
- Verify auto-scaling triggers at appropriate thresholds

**Latency Testing:**
- Measure end-to-end processing time on AMD Ryzen AI edge devices
- Verify OCR completes within 5 seconds
- Verify script generation completes within 15 seconds
- Verify total pipeline completes within 30 seconds (cloud) or 60 seconds (edge)

**Hardware Optimization Testing:**
- Verify NPU utilization on AMD Ryzen AI devices
- Verify multi-core parallelism on AMD EPYC processors
- Compare performance with and without hardware acceleration
- Measure power consumption on edge devices

### Security Testing

**Authentication and Authorization:**
- Test JWT token validation and expiration
- Test rate limiting enforcement
- Test row-level security policies in database

**Encryption Testing:**
- Verify bcrypt password hashing with 12+ rounds
- Verify TLS 1.3 enforcement for all API calls
- Verify AES-256 encryption for offline content

**Penetration Testing:**
- SQL injection attempts
- XSS attacks on web frontend
- CSRF token validation
- File upload vulnerabilities

### Test Coverage Goals

- Unit test coverage: Minimum 80% code coverage
- Property test coverage: All 25 correctness properties implemented
- Integration test coverage: All critical user flows
- Performance test coverage: All latency and throughput requirements
- Security test coverage: All authentication and encryption requirements

### Continuous Integration

**Automated Test Execution:**
- Run unit tests on every commit
- Run property tests on every pull request
- Run integration tests nightly
- Run performance tests weekly
- Run security scans on every release

**Test Environment:**
- Docker containers for consistent test environment
- Mock AMD hardware for CI/CD pipeline
- Dedicated test database with sample data
- Isolated object storage for test artifacts

### AMD Hardware-Specific Testing

**NPU Acceleration Validation:**
- Verify models load correctly on AMD Ryzen AI NPU
- Compare inference speed: NPU vs CPU
- Verify quantized models maintain accuracy within 5% of full models

**EPYC Multi-Core Validation:**
- Verify worker processes utilize available cores
- Measure throughput scaling with core count
- Verify no resource contention under load

**ROCm GPU Validation:**
- Verify models load correctly with ROCm
- Compare inference speed: ROCm GPU vs CPU
- Verify memory management on AMD GPUs
