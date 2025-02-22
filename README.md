# BSP
main.py - Entry point:
from app import app
if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
app.py - Flask application:
import os
import logging
import cv2
import numpy as np
from flask import Flask, render_template, request, jsonify
from werkzeug.utils import secure_filename
import base64
logging.basicConfig(level=logging.DEBUG)
app = Flask(__name__)
app.secret_key = os.environ.get("SESSION_SECRET", "default-secret-key")
ALLOWED_EXTENSIONS = {'png', 'jpg', 'jpeg'}
def allowed_file(filename):
    return '.' in filename and filename.rsplit('.', 1)[1].lower() in ALLOWED_EXTENSIONS
@app.route('/')
def index():
    return render_template('index.html')
@app.route('/upload', methods=['POST'])
def upload_file():
    if 'file' not in request.files:
        return jsonify({'error': 'No file part'}), 400
    file = request.files['file']
    if file.filename == '':
        return jsonify({'error': 'No selected file'}), 400
    if file and allowed_file(file.filename):
        try:
            filestr = file.read()
            np_img = np.frombuffer(filestr, np.uint8)
            image = cv2.imdecode(np_img, cv2.IMREAD_COLOR)
            image_rgb = cv2.cvtColor(image, cv2.COLOR_BGR2RGB)
            _, buffer = cv2.imencode('.jpg', image_rgb)
            image_base64 = base64.b64encode(buffer).decode('utf-8')
            return jsonify({
                'success': True,
                'image': f'data:image/jpeg;base64,{image_base64}',
                'width': image.shape[1],
                'height': image.shape[0]
            })
        except Exception as e:
            logging.error(f"Error processing image: {str(e)}")
            return jsonify({'error': 'Error processing image'}), 500
    return jsonify({'error': 'Invalid file type'}), 400
@app.route('/calculate_distance', methods=['POST'])
def calculate_distance():
    data = request.json
    point1 = data.get('point1')
    point2 = data.get('point2')
    scale_factor = data.get('scaleFactor', 1.0)
    if not point1 or not point2:
        return jsonify({'error': 'Invalid points'}), 400
    try:
        distance_pixels = np.sqrt(
            (point1['x'] - point2['x'])**2 + 
            (point1['y'] - point2['y'])**2
        )
        distance_real = distance_pixels * float(scale_factor)
        return jsonify({
            'success': True,
            'distance_pixels': float(distance_pixels),
            'distance_real': float(distance_real)
        })
    except Exception as e:
        logging.error(f"Error calculating distance: {str(e)}")
        return jsonify({'error': 'Error calculating distance'}), 500
utils/image_processing.py - Image processing utilities:
import cv2
import numpy as np
def process_image(image):
    gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
    blurred = cv2.GaussianBlur(gray, (5, 5), 0)
    _, thresh = cv2.threshold(blurred, 0, 255, cv2.THRESH_BINARY + cv2.THRESH_OTSU)
    contours, _ = cv2.findContours(thresh, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
    result = image.copy()
    cv2.drawContours(result, contours, -1, (0, 255, 0), 2)
    return result
def calculate_angle(point1, point2):
    dx = point2[0] - point1[0]
    dy = point2[1] - point1[1]
    angle = np.degrees(np.arctan2(dy, dx))
    return angle
def analyze_pattern_distribution(contours):
    pattern_metrics = {
        'centers': [],
        'areas': [],
        'distribution': {}
    }
    for contour in contours:
        M = cv2.moments(contour)
        if M['m00'] != 0:
            cx = int(M['m10'] / M['m00'])
            cy = int(M['m01'] / M['m00'])
            pattern_metrics['centers'].append((cx, cy))
        area = cv2.contourArea(contour)
        pattern_metrics['areas'].append(area)
    if pattern_metrics['centers']:
        centers = np.array(pattern_metrics['centers'])
        pattern_metrics['distribution'] = {
            'mean_x': np.mean(centers[:, 0]),
            'mean_y': np.mean(centers[:, 1]),
            'std_x': np.std(centers[:, 0]),
            'std_y': np.std(centers[:, 1])
        }
    return pattern_metrics
templates/index.html - Frontend template:
<!DOCTYPE html>
<html lang="en" data-bs-theme="dark">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Blood Spatter Pattern Analyzer</title>
    <link rel="stylesheet" href="https://cdn.replit.com/agent/bootstrap-agent-dark-theme.min.css">
    <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
</head>
<body>
    <div class="container py-4">
        <header class="pb-3 mb-4 border-bottom">
            <h1 class="display-5 fw-bold">Blood Spatter Pattern Analyzer</h1>
        </header>
        <div class="row">
            <div class="col-md-3">
                <div class="controls mb-4">
                    <h5 class="mb-3">Controls</h5>
                    
                    <div class="mb-3">
                        <label for="imageUpload" class="form-label">Upload Image</label>
                        <div class="upload-btn-wrapper">
                            <button class="btn btn-secondary w-100">Choose File</button>
                            <input type="file" id="imageUpload" accept=".jpg,.jpeg,.png" class="form-control">
                        </div>
                    </div>
                    <div class="mb-3">
                        <label for="scaleFactorInput" class="form-label">Scale Factor</label>
                        <input type="number" id="scaleFactorInput" class="form-control" 
                               value="1.0" step="0.1" min="0.1">
                        <div class="form-text">Units per pixel</div>
                    </div>
                    <button id="clearButton" class="btn btn-danger w-100">Clear Canvas</button>
                </div>
                <div class="measurement-display">
                    <h5 class="mb-3">Measurements</h5>
                    <p id="distanceOutput" class="mb-0">Distance: </p>
                </div>
            </div>
            <div class="col-md-9">
                <div class="canvas-container">
                    <canvas id="spatterCanvas"></canvas>
                </div>
                <div class="alert alert-info" role="alert">
                    <h5 class="alert-heading">Instructions:</h5>
                    <ol class="mb-0">
                        <li>Upload a blood spatter pattern image</li>
                        <li>Set the scale factor if known</li>
                        <li>Click two points to measure distance</li>
                        <li>Use clear button to reset measurements</li>
                    </ol>
                </div>
            </div>
        </div>
    </div>
    <script src="{{ url_for('static', filename='js/canvas.js') }}"></script>
</body>
</html>
static/js/canvas.js - Frontend JavaScript:
class BloodSpatterAnalyzer {
    constructor() {
        this.canvas = document.getElementById('spatterCanvas');
        this.ctx = this.canvas.getContext('2d');
        this.points = [];
        this.scaleFactor = 1.0;
        this.image = null;
        this.setupEventListeners();
    }
    setupEventListeners() {
        this.canvas.addEventListener('click', (e) => this.handleCanvasClick(e));
        document.getElementById('imageUpload').addEventListener('change', (e) => {
            this.handleImageUpload(e);
        });
        document.getElementById('scaleFactorInput').addEventListener('change', (e) => {
            this.scaleFactor = parseFloat(e.target.value) || 1.0;
        });
        document.getElementById('clearButton').addEventListener('click', () => {
            this.clearCanvas();
        });
    }
    async handleImageUpload(event) {
        const file = event.target.files[0];
        if (!file) return;
        const formData = new FormData();
        formData.append('file', file);
        try {
            const response = await fetch('/upload', {
                method: 'POST',
                body: formData
            });
            const data = await response.json();
            if (data.success) {
                this.loadImage(data.image, data.width, data.height);
            } else {
                alert(data.error || 'Error uploading image');
            }
        } catch (error) {
            console.error('Error:', error);
            alert('Error uploading image');
        }
    }
    loadImage(src, width, height) {
        this.image = new Image();
        this.image.onload = () => {
            this.canvas.width = width;
            this.canvas.height = height;
            this.ctx.drawImage(this.image, 0, 0);
            this.points = [];
        };
        this.image.src = src;
    }
    handleCanvasClick(event) {
        const rect = this.canvas.getBoundingClientRect();
        const x = event.clientX - rect.left;
        const y = event.clientY - rect.top;
        this.points.push({x, y});
        this.drawPoint(x, y);
        if (this.points.length === 2) {
            this.calculateDistance();
        }
    }
    drawPoint(x, y) {
        this.ctx.beginPath();
        this.ctx.arc(x, y, 5, 0, 2 * Math.PI);
        this.ctx.fillStyle = 'red';
        this.ctx.fill();
        this.ctx.stroke();
    }
    async calculateDistance() {
        try {
            const response = await fetch('/calculate_distance', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify({
                    point1: this.points[0],
                    point2: this.points[1],
                    scaleFactor: this.scaleFactor
                })
            });
            const data = await response.json();
            if (data.success) {
                document.getElementById('distanceOutput').textContent = 
                    `Distance: ${data.distance_real.toFixed(2)} units (${data.distance_pixels.toFixed(2)} pixels)`;
                this.ctx.beginPath();
                this.ctx.moveTo(this.points[0].x, this.points[0].y);
                this.ctx.lineTo(this.points[1].x, this.points[1].y);
                this.ctx.strokeStyle = 'yellow';
                this.ctx.stroke();
            }
            this.points = [];
        } catch (error) {
            console.error('Error:', error);
            alert('Error calculating distance');
        }
    }
    clearCanvas() {
        this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
        if (this.image) {
            this.ctx.drawImage(this.image, 0, 0);
        }
        this.points = [];
        document.getElementById('distanceOutput').textContent = 'Distance: ';
    }
}
document.addEventListener('DOMContentLoaded', () => {
    window.spatterAnalyzer = new BloodSpatterAnalyzer();
});
