<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Météo en Temps Réel</title>

    <!-- Leaflet.js CSS -->
    <link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css" />
    
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; font-family: Arial, sans-serif; }
        body, html { height: 100%; overflow: hidden; }
        #map { height: 100%; width: 100%; }

        /* Barre de recherche */
        #search-bar {
            position: absolute; top: 10px; left: 50%;
            transform: translateX(-50%);
            width: 90%; max-width: 300px;
            z-index: 1001;
        }
        input {
            width: 100%; padding: 8px; font-size: 12px;
            border: none; border-radius: 8px; outline: none;
        }
        /* Suggestions */
        #suggestions {
            position: absolute; width: 100%;
            background: white; max-height: 150px; overflow-y: auto;
            border-radius: 8px; box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
            display: none; z-index: 1002;
        }
        .suggestion {
            padding: 8px; cursor: pointer;
            border-bottom: 1px solid #ddd;
        }
        .suggestion:hover { background: #f0f0f0; }

        /* Bandeau météo */
        #info {
            position: absolute; bottom: 10px; left: 50%;
            transform: translateX(-50%);
            width: 90%; max-width: 300px;
            background: white; padding: 10px;
            border-radius: 15px; box-shadow: 0px 4px 10px rgba(0, 0, 0, 0.2);
            text-align: center; font-size: 14px;
            z-index: 1000;
        }
    </style>
</head>
<body>

    <!-- Barre de recherche avec suggestions -->
    <div id="search-bar">
        <input type="text" id="search" placeholder="Entrez une ville" oninput="getSuggestions()">
        <div id="suggestions"></div>
    </div>

    <!-- Carte -->
    <div id="map"></div>

    <!-- Infos météo -->
    <div id="info">
        <p id="location">📍 Lieu : --</p>
        <p id="temperature">🌡️ Température : -- °C</p>
        <p id="rain">🌧️ Pluie : -- mm</p>
        <p id="wind">💨 Vent : -- km/h (--°)</p>
    </div>

    <!-- Leaflet.js -->
    <script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>
    
    <script>
        const apiKey = "ecd2b9ec5a987fd2421a072328e82e86";
        let map, rainLayer, cloudLayer;

        function initMap(lat, lon, cityName = "Votre position") {
            if (!map) {
                map = L.map('map', { zoomControl: false }).setView([lat, lon], 10);
            } else {
                map.setView([lat, lon], 10);
            }

            L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
                attribution: '© OpenStreetMap'
            }).addTo(map);

            if (cloudLayer) map.removeLayer(cloudLayer);
            if (rainLayer) map.removeLayer(rainLayer);

            cloudLayer = L.tileLayer(`https://tile.openweathermap.org/map/clouds_new/{z}/{x}/{y}.png?appid=${apiKey}`, {
                attribution: '© OpenWeatherMap',
                opacity: 0.7
            }).addTo(map);

            rainLayer = L.tileLayer(`https://tile.openweathermap.org/map/precipitation_new/{z}/{x}/{y}.png?appid=${apiKey}`, {
                attribution: '© OpenWeatherMap',
                opacity: 0.5
            }).addTo(map);

            getWeatherData(lat, lon, cityName);
        }

        function getWeatherData(lat, lon, cityName) {
            fetch(`https://api.openweathermap.org/data/2.5/weather?lat=${lat}&lon=${lon}&units=metric&lang=fr&appid=${apiKey}`)
                .then(response => response.json())
                .then(data => {
                    document.getElementById("location").innerText = `📍 Lieu : ${cityName}`;
                    document.getElementById("temperature").innerText = `🌡️ Température : ${data.main.temp} °C`;
                    let rain = data.rain?.["1h"] || 0;
                    document.getElementById("rain").innerText = `🌧️ Pluie : ${rain} mm`;
                    document.getElementById("wind").innerText = `💨 Vent : ${(data.wind.speed * 3.6).toFixed(1)} km/h (${data.wind.deg}°)`;

                    updateRainColor(rain);
                })
                .catch(error => console.error("Erreur récupération météo", error));
        }

        function updateRainColor(rain) {
            let opacity = 0.3, hue = 90; // Vert par défaut

            if (rain >= 2 && rain < 5) { opacity = 0.5; hue = 60; } // Jaune
            else if (rain >= 5 && rain < 10) { opacity = 0.7; hue = 30; } // Orange
            else if (rain >= 10) { opacity = 0.9; hue = 0; } // Rouge

            rainLayer.setOpacity(opacity);
            cloudLayer.getContainer().style.filter = `hue-rotate(${hue}deg)`;
        }

        function getSuggestions() {
            let query = document.getElementById("search").value;
            if (query.length < 2) {
                document.getElementById("suggestions").style.display = "none";
                return;
            }

            fetch(`https://api.openweathermap.org/geo/1.0/direct?q=${query}&limit=5&appid=${apiKey}`)
                .then(response => response.json())
                .then(data => {
                    let suggestionsBox = document.getElementById("suggestions");
                    suggestionsBox.innerHTML = "";
                    if (data.length > 0) {
                        data.forEach(city => {
                            let div = document.createElement("div");
                            div.className = "suggestion";
                            div.innerText = `${city.name}, ${city.country}`;
                            div.onclick = () => selectCity(city.lat, city.lon, city.name);
                            suggestionsBox.appendChild(div);
                        });
                        suggestionsBox.style.display = "block";
                    } else {
                        suggestionsBox.style.display = "none";
                    }
                })
                .catch(error => console.error("Erreur de recherche", error));
        }

        function selectCity(lat, lon, cityName) {
            document.getElementById("search").value = cityName;
            document.getElementById("suggestions").style.display = "none";
            initMap(lat, lon, cityName);
        }

        window.onload = () => {
            navigator.geolocation.getCurrentPosition(
                (pos) => initMap(pos.coords.latitude, pos.coords.longitude),
                () => initMap(48.8566, 2.3522, "Paris") // Paris par défaut
            );
        };
    </script>

</body>
</html>
