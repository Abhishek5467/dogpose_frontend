---

# Dog Pose Detection IoT System

This project demonstrates an end-to-end IoT system that collects sensor data and performs dog pose detection via image uploads. The system consists of:

- **Backend:** A Flask server that:
  - Receives sensor data from an ESP32 device.
  - Simulates sensor data if no updates come within 1 minute.
  - Provides REST API endpoints (GET/POST) for sensor data.
  - Processes image uploads (via multipart/form-data) for dog pose prediction using a TensorFlow Lite model.
  
- **Frontend:** A React Native app that:
  - Fetches sensor data from the backend and displays it.
  - Allows users to capture or select images and upload them to get a pose prediction.

- **ESP32:** An Arduino (ESP32) based device that reads sensor values (e.g., temperature, humidity, motion) and sends them as JSON payloads via HTTP/HTTPS POST requests to the Flask server.

---

## Table of Contents

- [Architecture and Data Flow](#architecture-and-data-flow)
- [Protocols Used](#protocols-used)
  - [HTTP/HTTPS and REST API](#httphttps-and-rest-api)
  - [Multipart/Form-Data](#multipartform-data)
  - [CORS (Cross-Origin Resource Sharing)](#cors-cross-origin-resource-sharing)
- [TensorFlow Lite Model and Image Processing](#tensorflow-lite-model-and-image-processing)
- [Project Setup](#project-setup)
  - [Backend Setup](#backend-setup)
  - [Frontend Setup](#frontend-setup)
  - [ESP32 Setup](#esp32-setup)
- [Usage](#usage)
- [License](#license)

---

## Architecture and Data Flow

1. **Sensor Data Collection:**
   - **ESP32 Device:**  
     Reads sensor data (e.g., temperature, humidity, motion) and sends it in JSON format via a POST request to the Flask backend’s `/sensor_data` endpoint.
   - **Flask Backend:**  
     When sensor data is received, it updates an internal `sensor_data` dictionary and resets the `last_updated` timestamp. If no new data arrives within 1 minute, a background thread automatically generates simulated sensor data.

2. **Image Upload and Pose Prediction:**
   - **Mobile Frontend (React Native):**  
     The user selects or captures an image and uploads it via a multipart/form-data POST request to the `/predict` endpoint.
   - **Flask Backend Image Processing:**  
     The server uses Pillow to open, process, and resize the image. It then normalizes and feeds the image into a TensorFlow Lite model. The output is postprocessed by calculating key features (like aspect ratio and vertical center) to classify the dog pose into categories such as "standing," "sitting," "lying down," or "running." The prediction is returned as a JSON response.
   
3. **Data Retrieval:**
   - **GET Endpoints:**  
     Clients (mobile app or web applications) can fetch:
     - Sensor data from the `/sensor_data` endpoint.
     - The latest prediction result from the `/latest` endpoint.

---

## Protocols Used

### HTTP/HTTPS and REST API

- **HTTP Methods:**  
  - **GET:** Retrieves sensor data (via `/sensor_data` GET) and prediction results (via `/latest` GET).
  - **POST:** Sends sensor data (via `/sensor_data` POST) and images for pose detection (via `/predict` POST).

- **HTTPS:**  
  - In production, communication between the ESP32, the mobile app, and the Flask server is secured using HTTPS, ensuring that data is transmitted safely and encrypted.

### multipart/form-data

- **Purpose:**  
  - The multipart/form-data encoding type is used when sending files (such as images) along with other form data in one HTTP request.
  
- **How It Works:**  
  - The payload is separated into parts by a boundary string.
  - Each part contains its own headers, such as Content-Disposition and Content-Type.
  - For example, when the mobile app sends an image, the FormData includes a part with:
    - `Content-Disposition: form-data; name="image"; filename="upload.jpg"`
    - `Content-Type: image/jpeg`
    - Followed by the actual binary data of the image.

### CORS (Cross-Origin Resource Sharing)

- **Purpose:**  
  - CORS is used to allow web or mobile clients that are hosted on a different domain (or port) to access the Flask API endpoints.
  
- **How It Works:**  
  - The Flask server uses the Flask-CORS package to automatically add CORS headers such as `Access-Control-Allow-Origin: *` so that browsers or mobile clients can make cross-origin requests without being blocked by the same-origin policy.

---

## TensorFlow Lite Model and Image Processing

1. **Model Loading:**  
   - During server startup, a TensorFlow Lite model (best_int8.tflite) is loaded via the TFLite interpreter.
   - The model’s input and output tensor details are fetched.

2. **Image Preprocessing:**  
   - The image is received as part of a multipart/form-data request.
   - Pillow (PIL) converts the image to RGB, corrects its orientation (if EXIF data is available), resizes it to 160x160, and normalizes pixel values (scaling them from 0–255 to 0–1).
   - The processed image is reshaped to include a batch dimension before being fed to the model.

3. **Inference:**  
   - The preprocessed image is set as the input tensor.
   - The model is run via `interpreter.invoke()`.
   - The model outputs keypoints (features) which are postprocessed.

4. **Pose Classification:**  
   - The output keypoints are analyzed by computing the aspect ratio and vertical center.
   - Based on thresholds:
     - Aspect ratio > 1.3 is interpreted as "standing".
     - Vertical center > 0.6 indicates "sitting".
     - Aspect ratio < 1.0 indicates "lying down".
     - Otherwise, the pose is classified as "running".
   - The prediction is returned as JSON.

---

## Project Setup

### Backend Setup

1. **Dependencies:**
   - Python (3.x)
   - Flask, Flask-CORS
   - TensorFlow (for TensorFlow Lite)
   - Pillow (PIL)
   - NumPy

2. **Installation:**
   ```bash
   pip install flask flask-cors tensorflow pillow numpy
   ```

3. **Project Structure:**
   ```
   ├── app.py
   ├── model/
   │   └── best_int8.tflite
   ├── templates/
   │   └── index.html
   └── README.md
   ```

4. **Run Backend:**
   ```bash
   python app.py
   ```

### Frontend Setup

1. **React Native Setup:**
   - Use Expo or React Native CLI.
   - In your React Native project, configure the backend URL (e.g., `https://dogpose-detector.onrender.com`).

2. **Key Functions:**
   - Fetch sensor data using Axios (GET `/sensor_data`).
   - Upload images using FormData (POST `/predict`).

3. **Run Frontend:**
   - Follow your usual React Native development process (e.g., using Expo).

### ESP32 Setup

1. **Arduino IDE:**
   - Install required libraries (WiFi, HTTPClient, DHT sensor library).
   - Configure WiFi SSID and password.
   - Set the server endpoint URL to the backend’s sensor_data endpoint (preferably over HTTPS).

2. **Upload Code:**
   - Use the provided Arduino IDE code to connect to WiFi, read sensor data, and send updates to the Flask backend.

---

## Usage

1. **Sensor Data Flow:**
   - The ESP32 sends sensor data every 5 seconds. If no update is received for 1 minute, the backend simulates sensor data.
   - Clients (mobile apps/browsers) fetch sensor data via `/sensor_data`.

2. **Pose Prediction Flow:**
   - The mobile app uploads an image to the `/predict` endpoint.
   - The backend processes the image, runs inference using the TFLite model, classifies the dog's pose, and returns the result.
   - The result is displayed in the mobile app.

3. **Latest Prediction:**
   - Clients can also retrieve the most recent prediction using the `/latest` endpoint.

---

## License

Include your project’s license information here (e.g., MIT License).

---

This README provides a comprehensive overview of the project, explains how the frontend and backend connect and communicate, and details the protocols and processing involved. Adjust the content as needed for your specific project setup and any additional features you implement.

---
Answer from Perplexity: pplx.ai/share
