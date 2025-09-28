# AI-Powered Appointment Scheduler Assistant(Backend) 

The **AI-Powered Appointment Scheduler Assistant** is a backend system that processes **natural language appointment requests** and extracts **structured details** such as:  

- **Intent** → book, cancel, reminder, or unrelated  
- **Entities** → date, time, department (e.g., dentist, doctor)  
- **Normalization** → standard ISO date/time with timezone  

It supports **raw text** as well as **files (PDF, DOCX, images)** with OCR.  

---
## Setup Instrucions
## ⚙️ Setup Instructions  

Follow these steps to get the **Appointment NLP API** up and running locally with Ngrok:

---

### 1) Clone the Repository  

```bash
git clone https://github.com/<your-username>/appointment-nlp-api.git
cd appointment-nlp-api
```

### 2️) Create and Activate Virtual Environment
```bash
python -m venv venv
venv\Scripts\activate (Windows based)
source venv/bin/activate(mac based)
```

### 3️) Install Dependencies

Install all required Python packages:
```bash
pip install -r requirements.txt
```
Dependencies include:

    fastapi, uvicorn → API framework and server

    easyocr, pymupdf, python-docx → File parsing and OCR

    torch, torchvision, torchaudio → Deep learning framework

    spacy, gliner, transformers, parsedatetime, dateparser → NLP & entity extraction

    joblib, scikit-learn → Model serialization & ML utilities

    python-multipart → File upload support in FastAPI

    pyngrok → Create public tunnels for local server

    numpy, pandas, python-dotenv, supabase → Utilities and backend integration

### 4️) Post-install Step (spaCy Model)

Download the English model required for spaCy:
```bash
python -m spacy download en_core_web_sm
```




## Architecture  

The API is designed as a **multi-step pipeline**, combining **AI models** with **rule-based guardrails** for robust performance.  

### Step 1: Input Handling  
- Accepts:  
  - **Raw text** via JSON body  
  - **Files** (PDF, DOCX, PNG, JPG) via upload  
- If image/PDF/DOCX → OCR/Document parsing is applied.  

**Why this design?**  
- Real-world appointment requests often come from emails, scans, or chat messages.  
- Supporting multiple formats increases usability.  

**Alternative considered**: Restricting to text-only input (faster but less practical).  

---

### Step 2: Intent Classification  
- Uses **TF-IDF + Logistic Regression** to classify requests into:  
  - `appointment_request`  
  - `cancellation`  
  - `reminder`  
  - `not_related`  

**Why Logistic Regression?**  
- Lightweight, fast to train, interpretable.  
- Works well with limited training data.  

**Alternatives**:  
- Deep learning classifiers (transformers, BERT) → more accurate but heavy and slow for API latency requirements.  

---

### Step 3: Entity Extraction  
- Uses **[GLiNER](https://huggingface.co/urchade/gliner_small)** for extracting `date`, `time`, and `department`.  
- Rule-based fallback for **departments** (`dentist`, `doctor`, `cardiologist`, etc.).  
- Regex cleanup for **time expressions** (e.g., `3.00 pm → 3:00 pm`).  

**Why GLiNER?**  
- General-purpose NER that works out of the box for multiple entity types.  
- Lightweight compared to large NER models.  

**Alternative**: spaCy NER – accurate but requires custom training for domain-specific entities.  

---

### Step 4: Normalization  
- Uses `parsedatetime` + `pytz` to convert date/time phrases (`next Friday at 3pm`) → standardized `YYYY-MM-DD` + `HH:MM`.  
- Applies **Asia/Kolkata** timezone.  
- Guardrails return `"needs_clarification"` if parsing fails.  

**Why parsedatetime?**  
- Flexible natural language date parsing.  
- Handles relative expressions (`tomorrow`, `next Monday`).  

**Alternative**: Duckling (Facebook) → very powerful but requires extra runtime and setup.  

---

### Step 5: Multi-Appointment Handling  
- Splits text into multiple sentences (`and also`, `then`, `also`).  
- Extracts entities + normalizes each independently.  

**Why?**  
- Users often request multiple appointments in a single query.  

**Alternative**: Treat whole text as one → simpler but fails when multiple bookings exist.  

---

### Step 6: Guardrails & Error Handling  
- Returns `"needs_clarification"` if:  
  - Missing/ambiguous **date/time/department**  
  - Parsing fails  
- Rejects unsupported file types (`400 Bad Request`).  

---

##  API Usage  

## How the API Works  

The **Appointment NLP API** is built using **FastAPI** and served with **Uvicorn**.  
Here’s the flow from **input to response**:

---

### 🔹 1. User Input  
- The client (user or frontend) sends an HTTP `POST` request to `/process_appointment`.  
- Input can be:
  - **Text only** (JSON: `{"text": "Book a dentist appointment tomorrow at 3pm"}`)  
  - **Text + File** (multipart upload: text query + PDF/DOCX/Image)  

---

### 🔹 2. FastAPI Endpoint  
- FastAPI receives the request in `my_app.py`.  
- Example route:
  ```python
  @app.post("/process_appointment")
  async def process_appointment(text: str = Form(...), file: UploadFile = File(None)):
      ...

    Here:

        text = plain query string

        file = uploaded document or image

FastAPI automatically validates and parses inputs before they reach your function.

### 3. Processing Pipeline

Inside the endpoint, the request goes through:

    Preprocessing

        Extract text from file if provided (OCR for images, parser for PDF/DOCX).

        Combine with raw text input.

    Entity & Intent Extraction

        A model (e.g., GLiNER, spaCy, Duckling, or regex) extracts:

            Intent (book, cancel, reschedule, reminder)

            Entities (department, date, time, doctor)

    Normalization & Guardrails

        Dates are converted into ISO format (2025-10-05).

        Times normalized to 24-hour format (15:00).

        Extra checks ensure incomplete/invalid dates don’t break the pipeline.

    Response Construction

        JSON response is built:

        {
          "intent": "book",
          "entities": {
            "department": "dentist",
            "date": "2025-10-05",
            "time": "15:00"
          },
          "source": "text+file"
        }

### 4. FastAPI → Uvicorn

    FastAPI itself is just the web framework.

    To actually serve the API, we run Uvicorn, an ASGI server:uvicorn my_app:app --reload

    Here:

        my_app → Python file name (without .py)

        app → FastAPI app instance inside that file

    Uvicorn handles:

        HTTP requests/responses

        Async event loop

        High-performance serving

### 5. Ngrok (Optional Public URL)

    Locally, your app runs at http://127.0.0.1:8000.

    With Ngrok, you expose it as a public API like:

    https://cordilleran-creeded-janean.ngrok-free.dev/process_appointment



### Endpoint  
`POST /process_appointment`  
Handles  Multiple Appoinment case
---

### 1) Text Input  

**Request:**  
```bash
curl -X 'POST' \
  'https://cordilleran-creeded-janean.ngrok-free.dev/process_appointment' \
  -H 'accept: application/json' \
  -H 'Content-Type: multipart/form-data' \
  -F 'text=book dentist next week @ 3pm' \
  -F 'file=@image_test2.png;type=image/png'
Response Body:
  {
  "step1": {
    "raw_text": "book dentist next week at 3pm",
    "confidence": 0.95
  },
  "step2_intent": {
    "intent": "appointment_request",
    "intent_confidence": 0.39
  },
  "step3_entities": [
    {
      "entities": {
        "date_phrase": "next week",
        "time_phrase": "3pm",
        "department": "dentist"
      },
      "entities_confidence": 0.9
    }
  ],
  "step4_normalization": [
    {
      "normalized": {
        "date": "2025-10-05",
        "time": "15:00",
        "tz": "Asia/Kolkata"
      },
      "normalization_confidence": 0.95,
      "status": "ok"
    }
  ],
  "step5_final": {
    "appointments": [
      {
        "department": "Dentist",
        "date": "2025-10-05",
        "time": "15:00",
        "tz": "Asia/Kolkata"
      }
    ],
    "status": "ok"
  }
  ```

### 2) File Input(In this case Image)  

**Request:**  
```bash
curl -X 'POST' \
  'https://cordilleran-creeded-janean.ngrok-free.dev/process_appointment' \
  -H 'accept: application/json' \
  -H 'Content-Type: multipart/form-data' \
  -F 'text=' \
  -F 'file=@image_test2.png;type=image/png'

Response Body:
{
  "step1": {
    "raw_text": "Hip would like to book a dentist appointment for next Friday around 3:30 pmz and also set a reminder for my follow up cardiologist visit two weeks from now at 10 am. Please make sure the appointments do not clash with any existing schedule have.",
    "confidence": 0.77
  },
  "step2_intent": {
    "intent": "appointment_request",
    "intent_confidence": 0.31
  },
  "step3_entities": [
    {
      "entities": {
        "date_phrase": "next Friday",
        "time_phrase": "3:30 pm",
        "department": "dentist"
      },
      "entities_confidence": 0.9
    },
    {
      "entities": {
        "date_phrase": "two weeks from now",
        "time_phrase": "10 am",
        "department": "cardiologist"
      },
      "entities_confidence": 0.9
    },
    {
      "entities": {
        "date_phrase": null,
        "time_phrase": null,
        "department": null
      },
      "entities_confidence": 0.9
    }
  ],
  "step4_normalization": [
    {
      "normalized": {
        "date": "2025-10-03",
        "time": "15:30",
        "tz": "Asia/Kolkata"
      },
      "normalization_confidence": 0.95,
      "status": "ok"
    },
    {
      "normalized": {
        "date": "2025-10-12",
        "time": "10:00",
        "tz": "Asia/Kolkata"
      },
      "normalization_confidence": 0.95,
      "status": "ok"
    },
    {
      "status": "needs_clarification",
      "message": "Ambiguous date/time or department"
    }
  ],
  "step5_final": {
    "appointments": [
      {
        "department": "Dentist",
        "date": "2025-10-03",
        "time": "15:30",
        "tz": "Asia/Kolkata"
      },
      {
        "department": "Cardiologist",
        "date": "2025-10-12",
        "time": "10:00",
        "tz": "Asia/Kolkata"
      }
    ],
    "status": "ok"
  }
}
```
