from flask import Flask, request, render_template_string
import requests

app = Flask(__name__)

# Updated UI Design with Map, Distance, and Detailed Fields (Aapki Image aur Purane Map Code Dono ko Milakar)
HTML_TEMPLATE = """
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Live Number Tracker</title>
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"/>
    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
    <style>
        body { font-family: Arial, sans-serif; background: #0f172a; color: white; margin: 0; padding: 10px; display: flex; flex-direction: column; align-items: center; }
        .container { width: 100%; max-width: 440px; margin: auto; padding: 10px; box-sizing: border-box; }
        .card { background: #1e293b; padding: 20px; border-radius: 20px; margin-top: 15px; border: 1px solid #334155; box-sizing: border-box; }
        input, button { width: 100%; padding: 12px; margin-top: 10px; border-radius: 8px; border: none; box-sizing: border-box; font-size: 16px; }
        input { background: #334155; color: white; outline: none; }
        button { background: #3b82f6; color: white; font-weight: bold; cursor: pointer; }
        button:hover { background: #2563eb; }
        .title { font-size: 22px; font-weight: bold; margin-bottom: 15px; border-bottom: 1px solid #334155; padding-bottom: 10px; }
        .row { display: flex; justify-content: space-between; padding: 12px 0; border-bottom: 1px solid #2d3748; font-size: 16px; }
        .row:last-child { border-bottom: none; }
        .label { color: #94a3b8; font-weight: bold; }
        .value { color: #f1f5f9; text-align: right; max-width: 60%; word-wrap: break-word; }
        #map { height: 320px; border-radius: 15px; margin-top: 10px; border: 1px solid #334155; }
        #distance { margin-top: 10px; font-weight: bold; color: #3b82f6; text-align: center; }
    </style>
</head>
<body>

<div class="container">

    <div class="card">
        <form method="POST">
            <input type="text" name="number" placeholder="Enter 10 Digit Number" required>
            <button type="submit">Search Data</button>
        </form>
    </div>

    {% if data %}
    <div class="card">
        <div class="title">📋 Number Information</div>
        <div class="row"><div class="label">👤 Name:</div><div class="value">{{ data.name }}</div></div>
        <div class="row"><div class="label">🧑 Father Name:</div><div class="value">{{ data.father_name }}</div></div>
        <div class="row"><div class="label">📱 Mobile:</div><div class="value">{{ data.mobile }}</div></div>
        <div class="row"><div class="label">🏠 Address:</div><div class="value" id="addr">{{ data.address }}</div></div>
        <div class="row"><div class="label">📞 Alternate:</div><div class="value">{{ data.alternate }}</div></div>
        <div class="row"><div class="label">📡 Circle:</div><div class="value">{{ data.circle }}</div></div>
        <div class="row"><div class="label">🆔 Aadhaar:</div><div class="value">{{ data.aadhaar }}</div></div>
    </div>

    <div class="card">
        <div class="title">📍 Live Location Map</div>
        <div id="map"></div>
        <div id="distance">🗺️ Fetching your location to calculate distance...</div>
    </div>
    {% endif %}

</div>

<script>
// Map rendering logic (Sirf tab chalega jab data load hoga)
let address = document.getElementById("addr")?.innerText;

if (address && address !== "Address not found" && address !== "N/A") {
    let map = L.map('map').setView([20.59, 78.96], 5);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);

    navigator.geolocation.getCurrentPosition(function(pos) {
        let userLat = pos.coords.latitude;
        let userLng = pos.coords.longitude;

        L.marker([userLat, userLng]).addTo(map).bindPopup("You Are Here");

        // Geocoding the address using OpenStreetMap Nominatim API
        fetch("https://nominatim.openstreetmap.org/search?format=json&q=" + encodeURIComponent(address))
        .then(res => res.json())
        .then(d => {
            if (d.length > 0) {
                let lat = d[0].lat;
                let lon = d[0].lon;

                L.marker([lat, lon]).addTo(map).bindPopup("Target Location");
                L.polyline([[userLat, userLng], [lat, lon]], {color: "blue"}).addTo(map);
                map.fitBounds([[userLat, userLng], [lat, lon]]);

                // Haversine formula to calculate real distance in KM
                function dist(a, b, c, d) {
                    let R = 6371;
                    let dLat = (c - a) * Math.PI / 180;
                    let dLon = (d - b) * Math.PI / 180;
                    let x = Math.sin(dLat / 2) ** 2 + Math.cos(a * Math.PI / 180) * Math.cos(c * Math.PI / 180) * Math.sin(dLon / 2) ** 2;
                    return R * 2 * Math.atan2(Math.sqrt(x), Math.sqrt(1 - x));
                }

                let km = dist(userLat, userLng, lat, lon);
                document.getElementById("distance").innerHTML = "📏 Distance: " + km.toFixed(2) + " KM";
            } else {
                document.getElementById("distance").innerHTML = "⚠️ Address found but couldn't locate on Map.";
            }
        });
    }, function() {
        document.getElementById("distance").innerHTML = "❌ Please allow Location permission to see distance.";
    });
}
</script>

</body>
</html>
"""

def fetch_data(number):
    try:
        # 🟢 YAHAN RENDER WALI API CONNECT KI HAI 🟢
        url = f"https://tracexdata-api.onrender.com/api/lookup?number={number}"
        
        # Requesting data from the server
        res = requests.get(url, timeout=10)
        
        if res.status_code == 200:
            api_data = res.json()
            
            # API response se keys extract kar rahe hain
            return {
                "name": api_data.get("name", "N/A"),
                "father_name": api_data.get("father_name", "N/A"),
                "mobile": number,
                "address": api_data.get("address", "Address not found"),
                "alternate": api_data.get("alternate", "N/A"),
                "circle": api_data.get("circle", "N/A"),
                "aadhaar": api_data.get("aadhaar", "N/A")
            }
        else:
            return None
    except Exception as e:
        print("Backend Error:", e)
        return None

@app.route("/", methods=["GET", "POST"])
def home():
    data = None
    if request.method == "POST":
        number = request.form.get("number").strip()
        if number.isdigit() and len(number) == 10:
            data = fetch_data(number)
    return render_template_string(HTML_TEMPLATE, data=data)

if __name__ == "__main__":
    # Local port 5000 par run hoga
    app.run(host="0.0.0.0", port=5000, debug=True)
    
