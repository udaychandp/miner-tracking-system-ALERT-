``` python
import socket
import json
from flask import Flask, Response
from threading import Thread
import collections
import numpy as np
import math

# UDP Configuration
UDP_IP = "0.0.0.0"
UDP_PORT = 5010

# Flask App
app = Flask(__name__)

# Store distances for both anchors
distances = {"A1": 0.0, "A2": 0.0}

# EMA filter setup
EMA_ALPHA = 0.2  # Smoothing factor (0 < alpha < 1). Smaller alpha = more smoothing.
ema_values = {"A1": 0.0, "A2": 0.0}

# RSSI to Distance Parameters
MEASURED_POWER = -50  # RSSI at 1 meter
PATH_LOSS_EXPONENT = 3  # Typical for indoor/underground

def rssi_to_distance(rssi):
    if rssi is None or rssi == 0:
        return 0.0
    return (10 ** ((MEASURED_POWER - rssi) / (10 * PATH_LOSS_EXPONENT)))/10

# EMA filter function
def apply_ema_filter(anchor, new_distance):
    global ema_values
    if ema_values[anchor] == 0.0:  # Initialize with the first value
        ema_values[anchor] = new_distance
    else:
        ema_values[anchor] = (new_distance * EMA_ALPHA) + (ema_values[anchor] * (1 - EMA_ALPHA))
    return ema_values[anchor]

# HTML Content
index_html = """
<!DOCTYPE html>
<html>
<head>
    <title>Localization of Underground Mining</title>
    <style>
        canvas { border: 1px solid black; background-color: #f4f4f4; }
        .info { margin: 10px; font-size: 20px; }
    </style>
    <script>
        let distanceA1 = 0;
        let distanceA2 = 0;

        // Anchor positions in pixels (fixed on the map)
        const anchorA1_x = 100;
        const anchorA1_y = 300;
        const anchorA2_x = 500;
        const anchorA2_y = 300;
        const miner_y = 300; // Fixed Y position for the miner
        const pixel_to_meter_ratio = 20; // 1 meter = 20 pixels

        function updateDistances() {
            fetch("/distances")
                .then(response => response.json())
                .then(data => {
                    distanceA1 = data.A1;
                    distanceA2 = data.A2;
                    drawScene();
                });
        }

        function drawPillars(ctx) {
            ctx.fillStyle = "#000000";
            const pillarWidth = 80;
            const pillarHeight = 80;
            const spacing = 150;

            for (let i = 0; i < 4; i++) {
                let y = 30 + i * spacing;
                ctx.fillRect(30, y, pillarWidth, pillarHeight);
                ctx.fillRect(180, y, pillarWidth, pillarHeight);
                ctx.fillRect(330, y, pillarWidth, pillarHeight);
                ctx.fillRect(480, y, pillarWidth, pillarHeight);
            }
        }

        function drawMiner(ctx, x, y, color) {
            ctx.save();
            ctx.translate(x, y);
            ctx.lineWidth = 3;
            ctx.strokeStyle = color;
            ctx.fillStyle = color;

            // Miner drawing code...
            // Helmet
            ctx.beginPath();
            ctx.arc(0, -15, 8, Math.PI, 0);
            ctx.fill();
            // Head
            ctx.beginPath();
            ctx.arc(0, -10, 5, 0, 2 * Math.PI);
            ctx.fillStyle = "lightgray";
            ctx.fill();
            ctx.stroke();
            // Body
            ctx.beginPath();
            ctx.moveTo(0, -5);
            ctx.lineTo(0, 15);
            ctx.stroke();
            // Arms
            ctx.beginPath();
            ctx.moveTo(-5, 0);
            ctx.lineTo(5, 0);
            ctx.stroke();
            // Pickaxe handle
            ctx.beginPath();
            ctx.moveTo(5, 0);
            ctx.lineTo(15, -10);
            ctx.strokeStyle = "brown";
            ctx.stroke();
            // Pickaxe head
            ctx.beginPath();
            ctx.moveTo(12, -13);
            ctx.lineTo(18, -7);
            ctx.strokeStyle = "gray";
            ctx.stroke();
            // Legs
            ctx.beginPath();
            ctx.moveTo(0, 15);
            ctx.lineTo(-7, 25);
            ctx.moveTo(0, 15);
            ctx.lineTo(7, 25);
            ctx.strokeStyle = color;
            ctx.stroke();
            ctx.restore();
        }

        function drawScale(ctx) {
            ctx.fillStyle = "black";
            ctx.font = "12px Arial";
            ctx.fillText("0m", 100, 580);
            ctx.fillText("20m", 500, 580);
            ctx.beginPath();
            ctx.moveTo(100, 570);
            ctx.lineTo(500, 570);
            ctx.strokeStyle = "black";
            ctx.lineWidth = 2;
            ctx.stroke();
        }

        function drawScene() {
            const canvas = document.getElementById("tracker");
            const ctx = canvas.getContext("2d");

            // Clear the canvas
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // Draw static underground pillars
            drawPillars(ctx);
            
            // Draw Anchors
            ctx.fillStyle = "red";
            ctx.beginPath();
            ctx.arc(anchorA1_x, anchorA1_y, 10, 0, 2 * Math.PI);
            ctx.fill();
            
            ctx.fillStyle = "green";
            ctx.beginPath();
            ctx.arc(anchorA2_x, anchorA2_y, 10, 0, 2 * Math.PI);
            ctx.fill();

            // Draw the distance scale
            drawScale(ctx);

            // Calculate miner's X position based on inverse distance weighting
            // This time, the miner moves "away from" Anchor 1, toward Anchor 2.
            let miner_x = 0;
            if (distanceA1 > 0 && distanceA2 > 0) {
                const total_inverse_distance = 1/distanceA1 + 1/distanceA2;
                // The position is weighted to be closer to the anchor with the smaller distance.
                // This means a decreasing distance to A2 will move the miner to the right.
                miner_x = ( (1/distanceA2) * anchorA1_x + (1/distanceA1) * anchorA2_x ) / total_inverse_distance;
            } else if (distanceA1 > 0) {
                miner_x = anchorA1_x;
            } else if (distanceA2 > 0) {
                miner_x = anchorA2_x;
            } else {
                miner_x = (anchorA1_x + anchorA2_x) / 2;
            }
            
            // Draw the miner at the calculated position
            drawMiner(ctx, miner_x, miner_y, "blue");
            
            // Show distances text
            document.getElementById("distanceA1").innerText = "Anchor 1 Distance: " + distanceA1.toFixed(2) + " meters";
            document.getElementById("distanceA2").innerText = "Anchor 2 Distance: " + distanceA2.toFixed(2) + " meters";
        }

        window.onload = function() {
            setInterval(updateDistances, 500);
        };
    </script>
</head>
<body>
    <h2>Localization for Underground Mining</h2>
    <p class="info" id="distanceA1">Anchor 1 Distance: 0.00 meters</p>
    <p class="info" id="distanceA2">Anchor 2 Distance: 0.00 meters</p>
    <canvas id="tracker" width="600" height="600"></canvas>
</body>
</html>
"""

# Flask Routes
@app.route('/')
def index():
    return index_html

@app.route('/distances')
def get_distances():
    return json.dumps(distances)

# UDP Listener
def udp_listener():
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.bind((UDP_IP, UDP_PORT))
    print(f"Listening for RSSI packets on UDP port {UDP_PORT}...\n")
    
    try:
        while True:
            data, _ = sock.recvfrom(1024)
            try:
                msg = data.decode('utf-8', errors='ignore')
                parsed = json.loads(msg)
                anchor = parsed.get("anchor")
                rssi = parsed.get("rssi")
                distance = parsed.get("distance")
                payload = parsed.get("payload", "")

                if anchor in ["A1", "A2"]:
                    calculated_distance = 0.0
                    if distance is None or distance == "N/A":
                        calculated_distance = rssi_to_distance(rssi)
                    else:
                        calculated_distance = float(distance)
                    
                    # Apply EMA filter instead of moving average
                    filtered_distance = apply_ema_filter(anchor, calculated_distance)
                    distances[anchor] = filtered_distance
                    print(f"[{anchor}] RSSI: {rssi} dBm | Distance: {filtered_distance:.2f} m | Payload: {payload}")
                else:
                    print(f"Ignored packet from non-A1/A2 anchor: {anchor}")

            except json.JSONDecodeError:
                print("Invalid JSON received:", data.hex())
            except UnicodeDecodeError as e:
                print(f"Decoding error: {e}. Raw data: {data.hex()}")
            except ValueError as e:
                print(f"Value error in parsing distance: {e}. Raw data: {msg}")

    except KeyboardInterrupt:
        print("\nUDP listener stopped manually.")
    finally:
        sock.close()

# Run Flask and UDP listener in separate threads
if __name__ == '__main__':
    udp_thread = Thread(target=udp_listener)
    udp_thread.daemon = True
    udp_thread.start()
    
    app.run(host='0.0.0.0', port=5000)
```



# test 2 applied expo filter
``` python
import socket
import json
from flask import Flask, Response
from threading import Thread
import collections
import numpy as np
import math

# UDP Configuration
UDP_IP = "0.0.0.0"
UDP_PORT = 5010

# Flask App
app = Flask(__name__)

# Store distances for both anchors
distances = {"A1": 0.0, "A2": 0.0}

# EMA filter setup
EMA_ALPHA = 0.2  # Smoothing factor (0 < alpha < 1). Smaller alpha = more smoothing.
ema_values = {"A1": 0.0, "A2": 0.0}

# RSSI to Distance Parameters
MEASURED_POWER = -50  # RSSI at 1 meter
PATH_LOSS_EXPONENT = 3  # Typical for indoor/underground

def rssi_to_distance(rssi):
    if rssi is None or rssi == 0:
        return 0.0
    return 10 ** ((MEASURED_POWER - rssi) / (10 * PATH_LOSS_EXPONENT))

# EMA filter function
def apply_ema_filter(anchor, new_distance):
    global ema_values
    if ema_values[anchor] == 0.0:  # Initialize with the first value
        ema_values[anchor] = new_distance
    else:
        ema_values[anchor] = (new_distance * EMA_ALPHA) + (ema_values[anchor] * (1 - EMA_ALPHA))
    return ema_values[anchor]

# HTML Content
index_html = """
<!DOCTYPE html>
<html>
<head>
    <title>Localization of Underground Mining</title>
    <style>
        canvas { border: 1px solid black; background-color: #f4f4f4; }
        .info { margin: 10px; font-size: 20px; }
    </style>
    <script>
        let distanceA1 = 0;
        let distanceA2 = 0;

        // Anchor positions in pixels
        const anchorA1_x = 100;
        const anchorA1_y = 300;
        
        // Anchor 2 position is now defined relative to Anchor 1
        const anchorA2_x_relative = 400;
        const anchorA2_y_relative = 0;
        
        const miner_y = 300; // Fixed Y position for the miner
        const pixel_to_meter_ratio = 20; // 1 meter = 20 pixels

        function updateDistances() {
            fetch("/distances")
                .then(response => response.json())
                .then(data => {
                    distanceA1 = data.A1;
                    distanceA2 = data.A2;
                    drawScene();
                });
        }

        function drawPillars(ctx) {
            ctx.fillStyle = "#000000";
            const pillarWidth = 80;
            const pillarHeight = 80;
            const spacing = 150;

            for (let i = 0; i < 4; i++) {
                let y = 30 + i * spacing;
                ctx.fillRect(30, y, pillarWidth, pillarHeight);
                ctx.fillRect(180, y, pillarWidth, pillarHeight);
                ctx.fillRect(330, y, pillarWidth, pillarHeight);
                ctx.fillRect(480, y, pillarWidth, pillarHeight);
            }
        }

        function drawMiner(ctx, x, y, color) {
            ctx.save();
            ctx.translate(x, y);
            ctx.lineWidth = 3;
            ctx.strokeStyle = color;
            ctx.fillStyle = color;

            // Miner drawing code...
            // Helmet
            ctx.beginPath();
            ctx.arc(0, -15, 8, Math.PI, 0);
            ctx.fill();
            // Head
            ctx.beginPath();
            ctx.arc(0, -10, 5, 0, 2 * Math.PI);
            ctx.fillStyle = "lightgray";
            ctx.fill();
            ctx.stroke();
            // Body
            ctx.beginPath();
            ctx.moveTo(0, -5);
            ctx.lineTo(0, 15);
            ctx.stroke();
            // Arms
            ctx.beginPath();
            ctx.moveTo(-5, 0);
            ctx.lineTo(5, 0);
            ctx.stroke();
            // Pickaxe handle
            ctx.beginPath();
            ctx.moveTo(5, 0);
            ctx.lineTo(15, -10);
            ctx.strokeStyle = "brown";
            ctx.stroke();
            // Pickaxe head
            ctx.beginPath();
            ctx.moveTo(12, -13);
            ctx.lineTo(18, -7);
            ctx.strokeStyle = "gray";
            ctx.stroke();
            // Legs
            ctx.beginPath();
            ctx.moveTo(0, 15);
            ctx.lineTo(-7, 25);
            ctx.moveTo(0, 15);
            ctx.lineTo(7, 25);
            ctx.strokeStyle = color;
            ctx.stroke();
            ctx.restore();
        }

        function drawScale(ctx) {
            ctx.fillStyle = "black";
            ctx.font = "12px Arial";
            ctx.fillText("0m", 100, 580);
            ctx.fillText("20m", 500, 580);
            ctx.beginPath();
            ctx.moveTo(100, 570);
            ctx.lineTo(500, 570);
            ctx.strokeStyle = "black";
            ctx.lineWidth = 2;
            ctx.stroke();
        }

        function drawScene() {
            const canvas = document.getElementById("tracker");
            const ctx = canvas.getContext("2d");

            // Clear the canvas
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // Draw static underground pillars
            drawPillars(ctx);
            
            // Draw Anchor 1 at the fixed origin
            ctx.fillStyle = "red";
            ctx.beginPath();
            ctx.arc(anchorA1_x, anchorA1_y, 10, 0, 2 * Math.PI);
            ctx.fill();
            
            // Draw Anchor 2 relative to Anchor 1
            const anchorA2_x = anchorA1_x + anchorA2_x_relative;
            const anchorA2_y = anchorA1_y + anchorA2_y_relative;
            ctx.fillStyle = "green";
            ctx.beginPath();
            ctx.arc(anchorA2_x, anchorA2_y, 10, 0, 2 * Math.PI);
            ctx.fill();

            // Draw the distance scale
            drawScale(ctx);
            
            // Calculate total relative distance between anchors in meters
            const totalDistance = Math.sqrt(anchorA2_x_relative**2 + anchorA2_y_relative**2) / pixel_to_meter_ratio;
            
            // Calculate the miner's position based on a ratio of distances
            let miner_x = 0;
            if (distanceA1 > 0 && distanceA2 > 0) {
                 // Calculate the x-position based on the law of cosines or a similar geometric principle
                 // Here we use a simplified linear interpolation based on the distance ratio
                 const ratio = distanceA1 / (distanceA1 + distanceA2);
                 miner_x = anchorA1_x + (anchorA2_x - anchorA1_x) * (1-ratio);
            } else if (distanceA1 > 0) {
                // If only A1 is detected, place the miner at A1's location
                miner_x = anchorA1_x;
            } else if (distanceA2 > 0) {
                // If only A2 is detected, place the miner at A2's location
                miner_x = anchorA2_x;
            } else {
                // If no anchors are detected, place the miner in the middle
                miner_x = (anchorA1_x + anchorA2_x) / 2;
            }
            
            // Draw the miner at the calculated position
            drawMiner(ctx, miner_x, miner_y, "blue");
            
            // Show distances text
            document.getElementById("distanceA1").innerText = "Anchor 1 Distance: " + distanceA1.toFixed(2) + " meters";
            document.getElementById("distanceA2").innerText = "Anchor 2 Distance: " + distanceA2.toFixed(2) + " meters";
        }

        window.onload = function() {
            setInterval(updateDistances, 500);
        };
    </script>
</head>
<body>
    <h2>Localization for Underground Mining</h2>
    <p class="info" id="distanceA1">Anchor 1 Distance: 0.00 meters</p>
    <p class="info" id="distanceA2">Anchor 2 Distance: 0.00 meters</p>
    <canvas id="tracker" width="600" height="600"></canvas>
</body>
</html>
"""

# Flask Routes
@app.route('/')
def index():
    return index_html

@app.route('/distances')
def get_distances():
    return json.dumps(distances)

# UDP Listener
def udp_listener():
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.bind((UDP_IP, UDP_PORT))
    print(f"Listening for RSSI packets on UDP port {UDP_PORT}...\n")
    
    try:
        while True:
            data, _ = sock.recvfrom(1024)
            try:
                msg = data.decode('utf-8', errors='ignore')
                parsed = json.loads(msg)
                anchor = parsed.get("anchor")
                rssi = parsed.get("rssi")
                distance = parsed.get("distance")
                payload = parsed.get("payload", "")

                if anchor in ["A1", "A2"]:
                    calculated_distance = 0.0
                    if distance is None or distance == "N/A":
                        calculated_distance = rssi_to_distance(rssi)
                    else:
                        calculated_distance = float(distance)
                    
                    filtered_distance = apply_ema_filter(anchor, calculated_distance)
                    distances[anchor] = filtered_distance
                    print(f"[{anchor}] RSSI: {rssi} dBm | Distance: {filtered_distance:.2f} m | Payload: {payload}")
                else:
                    print(f"Ignored packet from non-A1/A2 anchor: {anchor}")

            except json.JSONDecodeError:
                print("Invalid JSON received:", data.hex())
            except UnicodeDecodeError as e:
                print(f"Decoding error: {e}. Raw data: {data.hex()}")
            except ValueError as e:
                print(f"Value error in parsing distance: {e}. Raw data: {msg}")

    except KeyboardInterrupt:
        print("\nUDP listener stopped manually.")
    finally:
        sock.close()

# Run Flask and UDP listener in separate threads
if __name__ == '__main__':
    udp_thread = Thread(target=udp_listener)
    udp_thread.daemon = True
    udp_thread.start()
    
    app.run(host='0.0.0.0', port=5000)
    ```
