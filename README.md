---

# Dog Pose Detection IoT System

This project demonstrates an end-to-end IoT system that collects sensor data and performs dog pose detection via image uploads. The system consists of three major components:

- **Backend (Flask Server):**  
  - Receives sensor data from an ESP32 device.
  - Simulates sensor data if no update is received within 1 minute.
  - Provides REST API endpoints (GET/POST) for sensor data and image-based pose prediction.
  - Processes image uploads (using multipart/form-data) through a TensorFlow Lite model.

- **Frontend (React Native App):**  
  - Fetches sensor data from the backend and displays it.
  - Allows users to capture or select images and upload them for dog pose prediction.

- **ESP32 Device (Arduino IDE Code):**  
  - Reads sensor values (e.g., temperature, humidity, motion).
  - Sends sensor data as JSON payloads via HTTP/HTTPS POST requests to the Flask backend’s `/sensor_data` endpoint.

---

## Table of Contents

- [Architecture and Data Flow](#architecture-and-data-flow)
- [Protocols and Data Formats Used](#protocols-and-data-formats-used)
  - [HTTP/HTTPS and REST API](#httphttps-and-rest-api)
  - [Multipart/Form-Data](#multipartform-data)
  - [CORS (Cross-Origin Resource Sharing)](#cors-cross-origin-resource-sharing)
- [TensorFlow Lite Model and Image Processing](#tensorflow-lite-model-and-image-processing)
- [Project Setup](#project-setup)
  - [Backend Setup](#backend-setup)
  - [Frontend Setup](#frontend-setup)
  - [ESP32 Setup (Arduino Code)](#esp32-setup-arduino-code)
- [Usage](#usage)
- [License](#license)

---

## Architecture and Data Flow

1. **Sensor Data Collection:**
   - **ESP32 Device:**  
     Reads sensor data (e.g., temperature, humidity, motion) from sensors and sends JSON via a POST request to the `/sensor_data` endpoint on the Flask server.
   - **Flask Backend:**  
     Updates an internal `sensor_data` dictionary with incoming values and resets the `last_updated` timestamp. If no new sensor data is received within 1 minute, a background thread simulates sensor data.
   - **Client Retrieval:**  
     Clients (frontend web or mobile apps) perform GET requests to `/sensor_data` to fetch current sensor data along with a freshness flag.

2. **Image Upload and Pose Prediction:**
   - **Mobile App (React Native):**  
     Captures or selects an image, then uploads it as multipart/form-data via POST to the `/predict` endpoint.
   - **Flask Backend Processing:**  
     - The uploaded image is processed: opened, correctly oriented using EXIF data, resized to 160×160 pixels, and normalized.
     - The preprocessed image is fed into a TensorFlow Lite model running via a TFLite interpreter.
     - The model outputs keypoints (features). Postprocessing computes an aspect ratio and vertical center.
     - Based on defined thresholds (e.g., for aspect ratio and vertical center), the dog’s pose is classified as "standing," "sitting," "lying down," or "running."
   - **Result:**  
     The prediction is returned as JSON and displayed in the mobile app.

3. **Data Retrieval:**
   - Clients can also fetch the latest prediction result using the `/latest` endpoint.

---

## Protocols and Data Formats Used

### HTTP/HTTPS and REST API

- **HTTP/HTTPS:**  
  The system uses RESTful API endpoints. Clients make GET and POST requests over HTTP or HTTPS (in production).  
  - **GET:** Used for retrieving sensor data and prediction results.
  - **POST:** Used for sending sensor data (JSON payload) and uploading images (multipart/form-data).

### Multipart/Form-Data

- **Purpose:**  
  Allows the mobile app to upload files (images) along with any textual data in a single POST request.
- **How It Works:**  
  - The payload is split into multiple “parts” by a boundary string defined in the `Content-Type` header.
  - Each “part” has its own headers (e.g., Content-Disposition, Content-Type) and contains the corresponding data (such as binary image data).
  - Flask automatically parses these parts into `request.files` (for file data) and `request.form` (for text fields).

### CORS (Cross-Origin Resource Sharing)

- **Purpose:**  
  Enables web or mobile clients hosted on different origins (domains, ports) to access the backend API.
- **How It Works:**  
  - The Flask-CORS package adds the necessary CORS headers (e.g., `Access-Control-Allow-Origin: *`).
  - This tells browsers and clients that it’s safe to perform cross-origin requests.

---

## TensorFlow Lite Model and Image Processing

1. **Model Loading and Inference:**
   - The TensorFlow Lite model (best_int8.tflite) is loaded at the start using the TFLite interpreter.
   - Input and output tensor details are retrieved for data preparation and reading inference results.
  
2. **Image Preprocessing:**
   - The uploaded image is read using Pillow (PIL) and converted to RGB.
   - EXIF data is used to correct image orientation if necessary.
   - The image is resized to 160×160 pixels, then converted into a normalized NumPy array and reshaped to add a batch dimension.

3. **Postprocessing & Pose Classification:**
   - The model outputs keypoints which are used to calculate features like the aspect ratio and vertical center.
   - Based on thresholds:
     - Aspect ratio > 1.3 → "standing"
     - Vertical center > 0.6 → "sitting"
     - Aspect ratio

```
#include <WiFi.h>
#include <HTTPClient.h>
#include <DHT.h>

// WiFi credentials
const char* ssid = "Infinix HOT 50 5G";
const char* password = "jaishriram";

// Server endpoint (use HTTPS if possible)
const char* serverUrl = "https://dogpose-detector.onrender.com/sensor_data";

// Sensor pins and configuration
#define DHTPIN 21
#define DHTTYPE DHT11
#define PIRPIN 22

DHT dht(DHTPIN, DHTTYPE);

// Variables for tracking state
unsigned long lastSendTime = 0;
const unsigned long SEND_INTERVAL = 5000;  // Send data every 5 seconds

void setup() {
  Serial.begin(115200);
  while (!Serial) delay(100);  // Wait for serial to initialize

  // Initialize sensors
  pinMode(PIRPIN, INPUT);
  dht.begin();

  // Connect to WiFi
  WiFi.begin(ssid, password);
  Serial.print("Connecting to WiFi");
  int attempts = 0;
  while (WiFi.status() != WL_CONNECTED && attempts = SEND_INTERVAL && WiFi.status() == WL_CONNECTED) {
    lastSendTime = currentMillis;
    
    // Read sensor data
    float temperature = dht.readTemperature();
    float humidity = dht.readHumidity();
    bool motionDetected = digitalRead(PIRPIN);
    
    // Check if readings are valid
    if (isnan(temperature) || isnan(humidity)) {
      Serial.println("Failed to read from DHT sensor!");
      return;
    }
    
    // Format data as JSON
    String payload = "{\"temperature\": " + String(temperature, 1) +
                     ", \"humidity\": " + String(humidity, 1) +
                     ", \"motion\": " + String(motionDetected ? "true" : "false") + "}";
    
    Serial.print("Sending data: ");
    Serial.println(payload);
    
    // Send HTTP POST request
    HTTPClient http;
    http.begin(serverUrl);
    http.addHeader("Content-Type", "application/json");
    
    int httpResponseCode = http.POST(payload);
    if (httpResponseCode > 0) {
      String response = http.getString();
      Serial.println("Server response code: " + String(httpResponseCode));
      Serial.println("Response: " + response);
    } else {
      Serial.print("Error sending data. HTTP error code: ");
      Serial.println(httpResponseCode);
    }
    
    http.end();
  }
}
```

---

## Usage

1. **Sensor Data Flow:**
   - **ESP32:** Sends sensor data every 5 seconds to the backend’s `/sensor_data`.
   - **Backend:** Updates its internal sensor data dictionary. If no update is received within 1 minute, the backend automatically simulates new data.
   - **Client:** Retrieves sensor data through a GET request to `/sensor_data`.

2. **Pose Prediction Flow:**
   - **Mobile App:** Captures or selects an image and uploads it via a POST request (multipart/form-data) to the `/predict` endpoint.
   - **Backend:** Processes the image (orientation, resizing, normalization), runs inference using the TensorFlow Lite model, and classifies the dog’s pose (standing, sitting, lying down, running) based on keypoint features.
   - **Result:** The prediction is returned as JSON and can be displayed in the mobile app.

3. **Latest Prediction:**
   - The `/latest` endpoint provides the most recent prediction result.

---

## License

Include your project’s license information here (e.g., MIT License).

---

## Additional Explanations

### What is a Payload in multipart/form-data?

In HTTP requests, the payload is the data in the body of the request. For multipart/form-data requests:
- The payload is broken down into multiple parts, each separated by a unique boundary string.
- Each part has headers, such as Content-Disposition and Content-Type, that explain the type of data included (e.g., a file or text).
- This format allows you to send multiple pieces of data (binary files mixed with form fields) in a single request.

### How does CORS Work?

CORS (Cross-Origin Resource Sharing) is a mechanism that allows web or mobile clients hosted on a different origin (domain, port, or scheme) to access resources on your Flask server. The Flask-CORS extension automatically adds headers like `Access-Control-Allow-Origin: *` so that modern browsers and clients can perform cross-origin HTTP requests without being blocked by the same-origin policy.

---
---
