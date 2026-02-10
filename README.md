# Indian Sign Language Recognition System

A comprehensive real-time **Indian Sign Language (ISL)** recognition system. The system uses computer vision, machine learning, and web technologies to provide real-time gesture recognition through a web interface. It supports both single-hand and two-hand gesture recognition with high accuracy.

## 📸 Dataset Overview

The system recognizes **35 Indian Sign Language classes** covering digits (1-9) and letters (A-Z). Below is a visual representation of all classes in the dataset:

<div align="center">

![Indian Sign Language Dataset - All 35 Classes](dataset_classes_visualization.png)

*Figure 1: Complete visualization of all 35 classes in the Indian Sign Language dataset. Each cell shows a sample image from the corresponding class folder.*

</div>

## 📋 Table of Contents

- [Features](#features)
- [Architecture](#architecture)
- [System Requirements](#system-requirements)
- [Installation](#installation)
- [Dataset](#dataset)
- [Model Architecture](#model-architecture)
- [Usage](#usage)
- [Project Structure](#project-structure)
- [API Documentation](#api-documentation)
- [Training](#training)
- [Performance](#performance)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)

## ✨ Features

### ISL Model Support
- **ISL Landmark Model**: Keras neural network using MediaPipe hand landmarks (35 classes: digits 1-9 + letters A-Z)
- **ISL Skeleton Model**: RandomForest classifier using normalized landmark features (35 classes)
- **Two-Hand Support**: Recognizes gestures from both single hand and two hands simultaneously

### Real-Time Recognition
- Live webcam feed processing
- Real-time gesture detection and classification
- WebSocket-based communication for instant updates
- Gesture stability detection to reduce false positives

### User Interface
- Modern web-based interface
- Real-time video streaming (MJPEG)
- Live prediction display
- Two-hand detection visualization
- Confidence score visualization

### Technical Features
- GPU acceleration support (CUDA)
- Multi-backend camera support (DirectShow, MSMF, V4L2)
- Robust error handling and camera reconnection
- Cross-platform compatibility (Windows, Linux, macOS)

## 🔍 How this project works

At a high level, **SignSpeak** turns your hand gestures into text and speech in six stages:

1. **Camera capture**
   - Your browser loads the `/app` page and displays an MJPEG video stream from `/video_feed`.
   - The backend (`app.py`) opens your webcam using OpenCV with Windows-friendly backends (DirectShow/MSMF) and continuously grabs frames.

2. **Hand detection with MediaPipe**
   - Each frame is passed through **MediaPipe Hands** (up to 2 hands).
   - For every detected hand, the model returns 21 landmarks (x, y positions of key joints).
   - Landmarks are drawn on the frame for visual feedback.

3. **Feature extraction (84 features)**
   - Landmarks are normalized relative to each hand’s bounding box.
   - For each hand, the 21 points are flattened into a **42‑element vector** (x and y for each point).
   - One-hand case: 42 features + 42 zeros = **84 features**.
   - Two-hand case: hand 1 (42) + hand 2 (42) = **84 features**.
   - This combined 84‑dimensional feature vector is shared by both models.

4. **Two-model ensemble prediction**
   - **ISL Skeleton model (RandomForest)** runs on the 84‑dimensional feature vector.
   - **ISL Keras landmark model (TensorFlow/Keras)** runs on the same 84 features.
   - A combiner function compares both predictions:
     - Prefers the Keras model (weighted confidence).
     - Falls back to the other model if one is unavailable.
   - The output is a predicted **class** (digits 1–9, letters A–Z) and **confidence**.

5. **Temporal smoothing + WebSocket emission**
   - Predictions are not emitted immediately; they are collected in a **sliding window** of recent frames.
   - When one class dominates the window (above a stability threshold) and confidence is high enough, that class is considered **stable**.
   - Stable predictions are:
     - Converted to display text (`"C"` → space, others → characters).
     - Emitted over **Socket.IO** as a `prediction` event with:
       - `text`, `confidence`, `model`, `num_hands`, `stability`, and raw `gesture`.
     - Logged to `prediction_log.csv` along with timestamp and user ID (if logged in).

6. **Frontend text building & speech**
   - The web client (`templates/index.html`):
     - Receives `prediction` events via Socket.IO.
     - Appends characters/words to a live paragraph, handling spaces and punctuation.
     - Updates a confidence bar and word-suggestion chips (driven by `static/dictionary.json`).
   - When you click **“Speak All”**:
     - The full paragraph is split **word by word**.
     - Browser speech synthesis speaks each word sequentially using your selected speed.

On top of this core pipeline, the app adds:

- **Authentication** (signup, login, logout) backed by SQLite (`app.db`) and session cookies.
- A modern, responsive UI for both the **landing page** and the **live recognition app**.

## 🏗️ Architecture

### System Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Web Browser (Client)                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   HTML/JS    │  │  WebSocket   │  │   MJPEG      │     │
│  │   Interface  │◄─┤   Socket.IO  │◄─┤   Stream     │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
                            ▲
                            │ HTTP/WebSocket
                            │
┌─────────────────────────────────────────────────────────────┐
│              Flask Application Server (app.py)               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Video Frame Generator                   │   │
│  │  ┌──────────────┐  ┌──────────────┐                │   │
│  │  │   Camera     │  │  MediaPipe   │                │   │
│  │  │   Capture    │─►│  Hand        │                │   │
│  │  │              │  │  Detection   │                │   │
│  │  └──────────────┘  └──────────────┘                │   │
│  │                          │                           │   │
│  │                          ▼                           │   │
│  │  ┌──────────────────────────────────────┐          │   │
│  │  │      ISL Model Prediction             │          │   │
│  │  │  ┌──────────┐      ┌──────────┐      │          │   │
│  │  │  │   ISL     │      │   ISL     │      │          │   │
│  │  │  │ Keras     │      │ Skeleton │      │          │   │
│  │  │  │ Landmark  │      │ Random   │      │          │   │
│  │  │  │ Model     │      │ Forest   │      │          │   │
│  │  │  └──────────┘      └──────────┘      │          │   │
│  │  └──────────────────────────────────────┘          │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Component Breakdown

#### 1. **Frontend (Web Interface)**
- **Technology**: HTML5, JavaScript, Socket.IO Client
- **Features**:
  - Real-time video display
  - Live prediction updates
  - Model switching controls
  - Responsive design

#### 2. **Backend Server (Flask)**
- **Technology**: Flask, Flask-SocketIO
- **Responsibilities**:
  - HTTP server for web interface
  - WebSocket server for real-time communication
  - MJPEG video streaming
  - Request routing and API endpoints

#### 3. **Computer Vision Pipeline**
- **MediaPipe Hands**: Hand landmark detection
- **OpenCV**: Image processing and camera management
- **Frame Processing**: Real-time frame capture and preprocessing

#### 4. **Machine Learning Models**

##### ISL Keras Landmark Model
- **Type**: Dense Neural Network (Keras/TensorFlow)
- **Input**: 84 normalized hand landmark features (42 per hand × 2 hands, padded with zeros if single hand)
- **Output**: 35 classes (digits 1-9 + letters A-Z)
- **Architecture**:
  ```
  Input (84 features)
    ↓
  Dense(256) + BatchNorm + Dropout(0.3)
    ↓
  Dense(128) + BatchNorm + Dropout(0.3)
    ↓
  Dense(64) + BatchNorm + Dropout(0.2)
    ↓
  Dense(35, softmax) → Output
  ```
- **File**: `model/indian_sign_model.h5` or `checkpoints/best_model_*.h5`
- **Accuracy**: 99.88% test accuracy
- **Preprocessing**:
  - Extract MediaPipe hand landmarks from all detected hands
  - Normalize coordinates relative to hand bounding box
  - Combine features from multiple hands (pad with zeros if single hand)
  - Create feature vector: 84 features (42 per hand)

##### ISL Skeleton Model (RandomForest)
- **Type**: RandomForest Classifier (scikit-learn)
- **Input**: 84 normalized hand landmark features (same format as Keras model)
- **Output**: 35 classes (digits 1-9 + letters A-Z)
- **File**: `model/model.p` (pickled scikit-learn model)
- **Preprocessing**:
  - Extract MediaPipe hand landmarks
  - Normalize coordinates relative to hand bounding box
  - Combine features from multiple hands (pad with zeros if single hand)
  - Create feature vector: 84 features

#### 5. **Prediction Pipeline**

```
Camera Frame
    ↓
MediaPipe Hand Detection (max 2 hands)
    ↓
Extract Landmarks from All Detected Hands
    ↓
┌─────────────────────────────────────┐
│   Feature Combination                │
│   - Single hand: 42 features +       │
│     42 zeros = 84 features          │
│   - Two hands: 42 + 42 = 84 features│
└─────────────────────────────────────┘
    ↓
┌─────────────────┬─────────────────┐
│   ISL Keras     │   ISL Skeleton  │
│   Landmark      │   RandomForest  │
│   Model         │   Model         │
│   (84 features) │   (84 features) │
└─────────────────┴─────────────────┘
    ↓
Ensemble Prediction (use best confidence)
    ↓
WebSocket Emission
    ↓
Frontend Display
```

## 💻 System Requirements

### Minimum Requirements
- **OS**: Windows 10/11, Linux (Ubuntu 18.04+), macOS 10.14+
- **Python**: 3.8 or higher
- **RAM**: 4 GB minimum (8 GB recommended)
- **Storage**: 2 GB free space
- **Camera**: USB webcam or built-in camera

### Recommended Requirements
- **CPU**: Multi-core processor (Intel i5/AMD Ryzen 5 or better)
- **GPU**: NVIDIA GPU with CUDA support (optional, for faster inference)
- **RAM**: 8 GB or more
- **Camera**: HD webcam (720p or higher)

### Software Dependencies
- Python 3.8+
- pip (Python package manager)
- Git (for cloning repository)

## 📦 Installation (Step-by-step)

#### Step 1: Clone the Repository

```bash
git clone https://github.com/Sujith8257/sign-to-text-and-speech.git
cd sign-to-text-and-speech
```

### Step 2: Create Virtual Environment

**Windows:**
```bash
python -m venv venv
venv\Scripts\activate
```

**Linux/macOS:**
```bash
python3 -m venv venv
source venv/bin/activate
```

### Step 3: Install Dependencies

```bash
pip install -r requirements.txt
```

### Step 4: First-time setup

This project now includes:

- **User authentication** (signup, login, logout) using SQLite (`app.db`)
- **Prediction logging** to a CSV file (`prediction_log.csv`) for later analysis

All required files are created automatically on first run; you do **not** need to create any tables or CSV files by hand.

### Step 5: Verify Installation (optional)

```bash
python -c "import cv2, mediapipe, flask, tensorflow; print('All dependencies installed successfully!')"
```

## 📊 Dataset

### Indian Sign Language Dataset

The system includes a dataset of **35 classes** covering:
- **Digits**: 1, 2, 3, 4, 5, 6, 7, 8, 9
- **Letters**: A through Z (26 letters)

**Dataset Structure:**
```
Indian/
├── 1/
│   ├── 1.jpg
│   ├── 2.jpg
│   └── ...
├── 2/
│   ├── 1.jpg
│   └── ...
├── A/
│   ├── 1.jpg
│   └── ...
└── ... (35 total classes)
```

**Dataset Statistics:**
- **Total Classes**: 35
- **Total Images**: ~42,510 images
- **Format**: JPG images
- **Organization**: One folder per class

**Visualization:**
The dataset visualization image (shown at the top of this README) displays a grid of sample images from all 35 classes, providing a quick overview of the sign language gestures the system can recognize.

## 🧠 Model Architecture

### ISL Keras Landmark Model

**Input Features:**
- Hand landmarks from MediaPipe (21 landmarks per hand)
- Each landmark has (x, y) coordinates normalized relative to hand bounding box
- Single hand: 42 features + 42 zeros = 84 features
- Two hands: 42 features (hand 1) + 42 features (hand 2) = 84 features

**Model Architecture:**

| Layer | Type | Units | Output Shape | Parameters |
|-------|------|-------|--------------|------------|
| Input | InputLayer | - | (84) | 0 |
| Dense1 | Dense | 256 | (256) | 21,760 |
| | BatchNorm | - | (256) | 1,024 |
| | Dropout(0.3) | - | (256) | 0 |
| Dense2 | Dense | 128 | (128) | 32,896 |
| | BatchNorm | - | (128) | 512 |
| | Dropout(0.3) | - | (128) | 0 |
| Dense3 | Dense | 64 | (64) | 8,256 |
| | BatchNorm | - | (64) | 256 |
| | Dropout(0.2) | - | (64) | 0 |
| Output | Dense | 35 | (35) | 2,275 |

**Total Parameters**: 66,979 trainable parameters

**Model Details:**
- **Algorithm**: Dense Neural Network (Keras/TensorFlow)
- **Classes**: 35 (digits 1-9 + letters A-Z)
- **Training**: Uses MediaPipe landmark data from images
- **Test Accuracy**: 99.88%
- **Validation Accuracy**: 100.00%
- **Inference Speed**: ~2-5 ms per prediction (CPU)

### ISL Skeleton Model (RandomForest)

**Input Features:**
- Same 84-feature format as Keras model
- Normalized landmark coordinates

**Model Details:**
- **Algorithm**: RandomForest Classifier (scikit-learn)
- **Classes**: 35 (digits 1-9 + letters A-Z)
- **Training**: Uses MediaPipe landmark data
- **Inference Speed**: ~1-2 ms per prediction
- **File**: `model/model.p`

**Training Configuration:**
- **Optimizer**: Adam (learning rate: 0.001)
- **Loss Function**: Categorical Crossentropy
- **Batch Size**: 64
- **Epochs**: 100 (with early stopping, patience=15)
- **Validation Split**: 20%
- **Test Split**: 10%
- **Checkpointing**: Best model saved every epoch, periodic checkpoints every 5 epochs

## 🧮 How the algorithm & training work

### Algorithm (inference) in short

- **Input**: Single RGB frame from webcam.
- **Hand landmarks**: MediaPipe Hands detects up to 2 hands and returns 21 landmarks per hand.
- **Feature vector**: Landmarks are normalized and flattened into an 84‑D vector:
  - Single hand: 42 features (x, y for 21 points) + 42 zeros.
  - Two hands: 42 features per hand → 42 + 42 = 84.
- **Model outputs**:
  - **Keras landmark model**: 35‑way softmax over \( \{1,\dots,9,A,\dots,Z\} \).
  - **RandomForest skeleton model**: 35‑class probability distribution over the same labels.
- **Ensemble logic**:
  - If only one model is available, use it directly.
  - If both are available, boost the Keras confidence and pick the class with higher (possibly weighted) confidence.
- **Temporal smoothing**:
  - Maintain a window of the last \(N\) predictions (default \(N = 5\)).
  - Let \(c^\*\) be the most frequent class in the window, with count \(k\).
  - Define **stability** as:
    \[
    \text{stability} = \frac{k}{N}
    \]
  - Only if \(\text{stability} \ge 0.6\) and confidence \(\ge 0.75\) do we emit \(c^\*\) to the frontend.

### Training pipeline (how the model was trained)

1. **Dataset preparation**
   - All images are stored under `Indian/` in 35 class folders (digits 1–9, letters A–Z).
   - For each image:
     - MediaPipe Hands extracts landmarks.
     - Landmarks are normalized and converted into the same 84‑D feature vector used at inference time.
   - The dataset is then split into:
     - **Training set**: ~70%
     - **Validation set**: 20%
     - **Test set**: 10%

2. **Model optimization objective**
   - The Keras model outputs a probability vector \(p \in \mathbb{R}^{35}\) (softmax).
   - Ground‑truth labels are one‑hot encoded vectors \(y\).
   - The **categorical cross‑entropy loss** minimized during training is:
     \[
     \mathcal{L} = -\frac{1}{N} \sum_{i=1}^{N} \sum_{c=1}^{35} y_{i,c} \log p_{i,c}
     \]
   - Optimization uses the **Adam** optimizer with learning rate \(10^{-3}\), plus early stopping and learning‑rate scheduling (see `train_indian_model.py`).

3. **Accuracy metrics**
   - **Top‑1 accuracy** (reported in this README) is:
     \[
     \text{Top‑1 Accuracy} = \frac{\text{# predictions where } \arg\max_c p_{i,c} = \text{true class}}{\text{total # predictions}}
     \]
   - **Top‑3 accuracy** (also computed during experiments) is:
     \[
     \text{Top‑3 Accuracy} = \frac{\text{# predictions where true class is in top‑3 probabilities}}{\text{total # predictions}}
     \]

4. **Reported performance (from final trained model)**
   - **Training accuracy**: ~99.96%
   - **Validation accuracy**: ~100.00%
   - **Test accuracy (Top‑1)**: ~99.88%
   - **Top‑3 accuracy**: 100.00%

These metrics are computed on the held‑out test split generated from the landmark dataset and match the values shown in the **Performance** section below.

## 🚀 Usage (End-to-end flow)

### 1. Start the backend

1. **Activate virtual environment** (if not already active):

   ```bash
   # Windows
   venv\Scripts\activate

   # Linux/macOS
   source venv/bin/activate
   ```

2. **Run the Flask + Socket.IO server**:

   ```bash
   python app.py
   ```

3. Wait until you see logs like:

   - `ISL Skeleton Model: [OK] Loaded`
   - `ISL Keras Landmark Model: [OK] Loaded` (if TensorFlow and model file are available)
   - `Running in Docker: False/True`
   - `Debug mode: True/False`

   and optionally camera backend messages such as:

   - `Trying to open camera at index 0 using backend: CAP_DSHOW`
   - `[OK] Camera opened successfully at index 0`

### 2. Open the landing page

1. In your browser, go to:

   ```
   http://localhost:5000
   ```

2. You will see the **SignSpeak landing page** (`templates/lander.html`) with:

   - A short explanation of the app
   - A CTA button like **“Sign in to try it”**

3. Click **“Sign in to try it”** to go to the **login** screen.

### 3. Create an account (signup)

1. On the login screen, click **“Create an account”** to open the signup page.
2. Fill in:

   - **Email address**
   - **Password** (minimum 6 characters)
   - **Confirm password**

3. Submit the form:

   - A new user is created in the local SQLite database (`app.db`, table `users`).
   - Passwords are stored as **hashed values** using `werkzeug.security.generate_password_hash`.
   - You are automatically logged in and redirected to the **main app** at `/app`.

> You only need to sign up once per email. On future visits you can just sign in.

### 4. Sign in and log out

- **Sign in**
  1. Open `http://localhost:5000/login`.
  2. Enter the email and password you used at signup.
  3. On success you are redirected to `/app` (main SignSpeak interface).

- **Log out**
  - From the `/app` page (`templates/index.html`), use the **Logout** button in the top-right header.
  - This calls the `/logout` route, clears the session and sends you back to the landing page.

> The `/app` route is **protected**: if you are not logged in it will redirect you to `/login?next=/app`.

### 5. Use the live recognition interface

Once you are logged in and on `/app`:

1. **Allow camera access** when your browser asks for permission.
2. In the left **camera panel** you will see:

   - A DSLR-style camera frame streaming from `/video_feed`.
   - A “Hand Tracking Active” status box.
   - Optional lighting hints if the scene is too dark.

3. In the right **text output panel** you will see:

   - A **timestamp** (current time and weekday).
   - A **paragraph area** where recognized text appears as you sign.
   - A **confidence meter** with color-coded bar (Excellent / Good / Fair).
   - **Word suggestions** chips that adapt as you build words.
   - Buttons:
     - **Speak All** – uses browser speech synthesis to read the entire recognized text, **word by word**.
     - **Pause / Resume** – controls speech playback.
     - **Clear** – clears the recognized text area.

4. **Make Indian Sign Language gestures** in front of the camera:

   - MediaPipe detects up to **two hands**.
   - Landmarks are processed and combined into **84 features**.
   - Both **Keras landmark model** and **RandomForest skeleton model** run, then a combined prediction is chosen.
   - A temporal smoothing window ensures only stable gestures are emitted to the frontend.

5. Each stable gesture:

   - Is sent to the browser over Socket.IO as a `prediction` event.
   - Is appended to the paragraph as text.
   - Updates the confidence bar and word suggestions.

6. Special mapping:

   - A specific gesture (based on left-hand open palm, mapped to class `C`) is treated as a **space** between words.

### 6. How predictions are stored (CSV logging)

For each **stable prediction** that is emitted to the client, the backend also writes a row to `prediction_log.csv` in the project root. This is useful for analytics, debugging and dataset building.

Each row contains:

- `timestamp` – server-side time of the event.
- `user_id` – ID of the logged-in user (from `app.db`), if available.
- `gesture` – the raw predicted class (e.g. `A`, `B`, `C`).
- `text_emitted` – what the frontend actually receives (`'A'` or space).
- `confidence` – model confidence for this gesture.
- `model` – which model “won” (`ISL-Keras` or `ISL-Skeleton`).
- `num_hands` – 0, 1 or 2.
- `stability` – how consistent the prediction was in the smoothing window (e.g. `80%`).

You can open `prediction_log.csv` in Excel, Google Sheets or use pandas:

```python
import pandas as pd
df = pd.read_csv("prediction_log.csv")
print(df.head())
```

### Model Usage

The system uses an ensemble approach:
- Both ISL Keras and ISL Skeleton models run predictions simultaneously
- The prediction with higher confidence is selected
- If both models agree, confidence is boosted
- Supports both single-hand and two-hand gestures

### Run on Another Windows PC

To run the project on a **different Windows computer** (e.g. copy folder to USB and run there):

- **With Docker:** Copy the project folder (including `model/indian_sign_model.h5`), install [Docker Desktop](https://www.docker.com/products/docker-desktop/) on the other PC, then double-click **`run-with-docker.bat`** or run `docker-compose up --build`. Open http://localhost:5000. *(Note: webcam may not work inside Docker on Windows.)*
- **With Python (camera works):** Copy the folder, install Python 3.11 on the other PC, then run **`run-with-python.bat`** (or create venv, `pip install -r requirements.txt`, `python app.py`). Open http://localhost:5000.

See **[RUN-ON-ANOTHER-PC.md](RUN-ON-ANOTHER-PC.md)** for step-by-step instructions and what to copy.

### Command Line Options

Currently, the application runs with default settings. Future versions may include:
- Custom port selection
- Camera index selection
- Model path specification
- Debug mode

## 📁 Project Structure

```
sign-to-text-and-speech/
│
├── app.py                          # Main Flask application
├── train_indian_model.py           # ISL CNN training script
├── inference_indian.py             # Standalone inference script
├── generate_dataset_visualization.py # Dataset visualization generator
│
├── requirements.txt                # Python dependencies
├── requirements_training.txt       # Additional training dependencies
│
├── model/                          # Model files directory
│   ├── indian_sign_model.h5       # ISL Keras landmark model
│   ├── model.p                    # ISL Skeleton RandomForest model
│   └── model_metadata.json       # Model metadata
│
├── checkpoints/                    # Training checkpoints
│   ├── best_model_*.h5            # Best model checkpoints
│   └── training_history_*.png     # Training history plots
│
├── Indian/                         # ISL Dataset
│   ├── 1/                         # Class folders
│   ├── 2/
│   ├── ...
│   ├── A/
│   ├── B/
│   └── ... (35 classes total)
│
├── templates/                     # Web templates
│   └── index.html                 # Main web interface
│
├── dataset_classes_visualization.png  # Dataset visualization
│
└── README.md                       # This file
```

## 🔌 API Documentation

### WebSocket Events

#### Server → Client

**`prediction`**
- **Purpose**: Send real-time prediction
- **Payload**:
  ```json
  {
    "text": "A",
    "confidence": 0.95,
    "model": "ISL-Keras" | "ISL-Skeleton",
    "num_hands": 1 | 2
  }
  ```

### HTTP Endpoints

**`GET /`**
- **Purpose**: Serve the landing page (`templates/lander.html`)
- **Response**: HTML page with marketing content and “Sign in” CTA

**`GET /login`**
- **Purpose**: Render login form
- **Response**: HTML page; on POST validates credentials and logs user in

**`GET /signup`**
- **Purpose**: Render signup form
- **Response**: HTML page; on POST creates a new user and logs them in

**`GET /logout`**
- **Purpose**: Clear the session and log the user out
- **Response**: Redirects to `/`

**`GET /app`**
- **Purpose**: Main protected web interface
- **Auth**: Requires valid session; otherwise redirects to `/login?next=/app`

**`GET /video_feed`**
- **Purpose**: MJPEG video stream
- **Response**: Multipart MJPEG stream
- **Content-Type**: `multipart/x-mixed-replace; boundary=frame`

**`GET /api/status`**
- **Purpose**: Get model status
- **Response**:
  ```json
  {
    "skeleton_model": true | false,
    "keras_model": true | false,
    "tensorflow": true | false
  }
  ```

## 🎓 Training

### Training the ISL Model

1. **Prepare Dataset**:
   - Ensure `Indian/` folder contains organized class folders
   - Each class folder should contain JPG images
   - Images will be processed to extract MediaPipe hand landmarks

2. **Configure Training**:
   - Edit `train_indian_model.py` to adjust:
     - Dataset path (automatically detected from common locations)
     - Model save paths (defaults to `model/` folder)
     - Batch size (default: 64), epochs (default: 100), learning rate (default: 0.001)
     - Two-hand support (default: True - uses 84 features)

3. **Run Training**:
   ```bash
   # Activate virtual environment first
   venv\Scripts\activate  # Windows
   # or
   source venv/bin/activate  # Linux/macOS
   
   python train_indian_model.py
   ```

4. **Monitor Training**:
   - Check console output for progress
   - Training automatically saves:
     - Best model: `checkpoints/best_model_*.h5`
     - Periodic checkpoints: `checkpoints/checkpoint_epoch_*.h5` (every 5 epochs)
     - Training history plot: `checkpoints/training_history_*.png`
   - Early stopping with patience=15 epochs
   - Learning rate reduction on plateau

5. **Use Trained Model**:
   - Final model saved to `model/indian_sign_model.h5`
   - Skeleton model saved to `model/model.p`
   - Metadata saved to `model/model_metadata.json`
   - Models are automatically loaded by `app.py`

### Training Parameters

**Default Settings:**
- **Batch Size**: 64
- **Epochs**: 100 (with early stopping, patience=15)
- **Learning Rate**: 0.001 (with ReduceLROnPlateau)
- **Input Features**: 84 (42 per hand × 2, padded if single hand)
- **Validation Split**: 20%
- **Test Split**: 10%
- **Two-Hand Support**: Enabled

**Checkpointing:**
- Best model saved every epoch (based on validation accuracy)
- Periodic checkpoints every 5 epochs
- Weight averaging from multiple checkpoints (optional)
- Automatic resume from latest checkpoint

**Expected Performance:**
- **Training Accuracy**: ~99.96%
- **Validation Accuracy**: ~100.00%
- **Test Accuracy**: ~99.88%
- **Top-3 Accuracy**: 100.00%

## 📈 Performance

### Model Performance

**ISL Keras Landmark Model:**
- **Test Accuracy**: 99.88%
- **Validation Accuracy**: 100.00%
- **Training Accuracy**: 99.96%
- **Top-3 Accuracy**: 100.00%
- **Inference Time**: 2-5 ms per frame (CPU)
- **Memory Usage**: ~260 KB (model) + ~50 MB (runtime)

**ISL Skeleton Model (RandomForest):**
- **Accuracy**: High accuracy (ensemble with Keras model)
- **Inference Time**: 1-2 ms per frame
- **Memory Usage**: ~5 MB

**Combined Ensemble:**
- Uses both models and selects prediction with highest confidence
- Typically achieves 99%+ accuracy in real-world scenarios

### System Performance

- **Frame Rate**: 15-30 FPS (depending on hardware)
- **Latency**: <100 ms end-to-end
- **CPU Usage**: 20-40% (single core)
- **GPU Usage**: 30-60% (if GPU available)

### Optimization Tips

1. **Use GPU**: Significantly faster inference for CNN model
2. **Reduce Image Size**: Lower resolution = faster processing
3. **Skip Frames**: Process every Nth frame for lower CPU usage
4. **Batch Processing**: Process multiple predictions together

## 🔧 Troubleshooting

### Camera Issues

**Problem**: Camera not detected
- **Solution**: Check camera permissions in browser/system settings
- **Alternative**: Try different camera index in code (0, 1, 2)

**Problem**: Camera opens but no frames
- **Solution**: Try different backend (DirectShow, MSMF, V4L2)
- **Check**: Camera is not being used by another application

### Model Loading Issues

**Problem**: Model file not found
- **Solution**: Ensure model files exist in project directory
- **Check**: File paths in `app.py` are correct

**Problem**: TensorFlow/Keras errors
- **Solution**: Reinstall TensorFlow: `pip install --upgrade tensorflow`
- **Check**: Python version compatibility (3.8+)

### Performance Issues

**Problem**: Low frame rate
- **Solution**: Reduce image resolution or skip frames
- **Alternative**: Use GPU acceleration if available

**Problem**: High CPU usage
- **Solution**: Reduce batch size or use GPU
- **Check**: Close other applications using CPU

### Web Interface Issues

**Problem**: Video stream not loading
- **Solution**: Check browser console for errors
- **Check**: Flask server is running and accessible

**Problem**: Predictions not updating
- **Solution**: Check WebSocket connection in browser console
- **Check**: Model is loaded correctly (check server logs)

## 🤝 Contributing

Contributions are welcome! Please follow these steps:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

### Areas for Contribution

- Additional sign language support (BSL, etc.)
- Model improvements and optimizations
- UI/UX enhancements
- Documentation improvements
- Bug fixes and performance optimizations
- Support for more complex gestures and phrases

## 📄 License

This project is licensed under the MIT License - see the LICENSE file for details.

## 🙏 Acknowledgments

- **MediaPipe**: For hand landmark detection
- **TensorFlow/Keras**: For deep learning framework
- **Flask**: For web framework
- **OpenCV**: For computer vision operations
- **scikit-learn**: For machine learning utilities

## 📧 Contact

For questions, issues, or contributions, please open an issue on GitHub.

---

**Last Updated**: December 2025
**Version**: 2.0.0

## 📝 Recent Updates

- **Removed ASL support** - Focused exclusively on Indian Sign Language (ISL)
- **New landmark-based model** - Replaced CNN with efficient landmark-based neural network
- **Two-hand support** - Full support for detecting and recognizing two-hand gestures
- **Improved accuracy** - Achieved 99.88% test accuracy with the new model
- **Model organization** - Models now organized in `model/` folder
- **Checkpoint system** - Enhanced checkpointing with weight averaging support
