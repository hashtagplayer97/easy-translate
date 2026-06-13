# EasyTranslate (Android AI Engine)

EasyTranslate is an ultra-low latency, fully offline, multi-modal AI translation engine built natively for Android. It is capable of running massive state-of-the-art Neural Machine Translation (NMT) and Speech-To-Text (STT) models locally on the device's edge hardware, circumventing the need for expensive API calls or internet connections.

## 📱 App Location
If you are looking to install the compiled application directly to your Android device, you can find the generated APK here:
`C:\Users\BHAVIK\Documents\app\app\build\outputs\apk\debug\app-debug.apk`

---

## 🏗️ Detailed Architecture

The architecture is explicitly designed to handle heavy Machine Learning models (1GB+) on memory-constrained mobile devices without triggering JVM Out-Of-Memory (OOM) crashes.

### 1. The Audio Engine (C++ / Oboe)
Android's JVM garbage collection causes micro-stutters which destroy real-time audio streams. To solve this, our audio layer is built purely in Native C++ using Google's **Oboe** library.
- **Lock-Free Ring Buffer**: A customized, thread-safe circular buffer (`LockFreeRingBuffer.cpp`) passes raw PCM audio data between the capture stream and the AI engines without ever blocking the CPU.

### 2. The ML inference Layer (Kotlin / JNI)
- **Whisper STT (TensorFlow Lite)**: Converts spoken audio into text.
- **NLLB-200 (ONNX Runtime)**: Facebook's "No Language Left Behind" engine, running in INT8 quantization via the ONNX C++ backend, capable of translating between 200 languages.
- **VITS TTS (ONNX Runtime)**: Generates synthesized speech from the translated text.
- **Grammar Engine**: An intermediary NLP layer to polish and humanize raw translation outputs.

### 3. Memory Management (`mmap()` System)
Because loading a 1.2GB PyTorch/ONNX model into a 4GB RAM phone will instantly crash it, the `ModelManager` bypasses the Java Heap entirely.
- It uses Java NIO to invoke the POSIX **`mmap()`** system call.
- The massive models are mapped directly into Virtual Memory. The Android Linux Kernel dynamically pages segments of the neural network in and out of physical RAM only when needed.
- A strict **180-second Garbage Collection Timer** aggressively unmaps (`munmap()`) the models if the user is idle, instantly returning gigabytes of RAM to the OS.

### 4. UI/UX Layer (Jetpack Compose)
A premium, multi-screen Glassmorphism UI built with Jetpack Navigation.
- **Modality Engine**: Allows seamless transition between 4 modes: Text-to-Text, Audio-to-Text, Text-to-Audio, and Audio-to-Audio.
- **Native Permissions**: Uses the direct OS Activity Results API for lightning-fast, highly accurate `RECORD_AUDIO` permissions without heavy third-party bloatware.

---

## ☁️ How to Connect the App to AWS (Model Hosting)

Because AI models are massive (1.5GB+), they cannot be packaged into the `.apk` file. They must be hosted on a cloud server like Amazon Web Services (AWS) and downloaded by the app at runtime. 

Here is the exact step-by-step pipeline to join the app with AWS:

### Step 1: Export and Convert Your Models
1. Download the raw NLLB-200, Whisper, and VITS models from Hugging Face.
2. Use Python (`optimum-cli`) to export them to quantized `.onnx` and `.tflite` formats.
   *Example:* `optimum-cli export onnx --model facebook/nllb-200-distilled-600M nllb_quantized/`

### Step 2: Setup AWS S3 (Simple Storage Service)
1. Log into your **AWS Console**.
2. Go to **S3** and click **Create Bucket** (e.g., `easytranslate-models-bucket`).
3. Uncheck "Block all public access" (or set up strict IAM Presigned URLs if you want to keep them private).
4. Upload your massive `.onnx` and `.tflite` files directly into this bucket.
5. Click on the uploaded file and copy its **Object URL** (e.g., `https://easytranslate-models-bucket.s3.amazonaws.com/nllb_model.onnx`).

### Step 3: Update `ModelManager.kt` in the App
Right now, the `ModelManager.kt` contains dummy URLs. You need to point it to your new AWS Bucket.
1. Open `app/src/main/java/com/easytranslate/app/ModelManager.kt`.
2. Locate the `downloadModel()` function.
3. Replace the placeholder URLs with your real AWS S3 URLs.
```kotlin
val awsUrl = "https://easytranslate-models-bucket.s3.amazonaws.com/${modelName}"
val request = Request.Builder().url(awsUrl).build()
```

### Step 4: Implement Tokenization (The Final ML Step)
Remember, Neural Networks do not understand strings (like "Hello"). They only understand mathematical IDs. 
1. You must host the `source.spm` and `target.spm` SentencePiece tokenizers on AWS as well.
2. The app will download them.
3. You will need to use a JNI wrapper to pass the user's text string into the SentencePiece library, which converts "Hello" to `[3412, 12]`. You then feed those numbers into the ONNX Runtime engine.

Once you upload the files to S3 and paste the links into `ModelManager.kt`, the "Language Manager" UI in the app will automatically handle downloading them to the user's phone with a real-time progress bar!
