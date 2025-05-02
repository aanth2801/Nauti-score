<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Nauti-Score</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
    <style>
        .nautical-bg {
            background: linear-gradient(to bottom, #1e3a8a, #60a5fa);
        }
        .spinner {
            border: 4px solid rgba(255, 255, 255, 0.3);
            border-top: 4px solid #ffffff;
            border-radius: 50%;
            width: 24px;
            height: 24px;
            animation: spin 1s linear infinite;
            display: none;
        }
        @keyframes spin {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }
        tr:hover {
            background-color: #e6f3ff;
        }
        #map {
            height: 400px;
            border-radius: 8px;
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
        }
    </style>
</head>
<body class="nautical-bg font-sans text-gray-800">
    <header class="bg-blue-900 text-white p-4 flex items-center justify-center">
        <svg class="w-8 h-8 mr-2" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 7l5 5m0 0l-5 5m5-5H6"></path>
        </svg>
        <h1 class="text-2xl font-bold">Nauti-Score</h1>
    </header>
    <div class="container mx-auto p-6 max-w-6xl">
        <div class="bg-white rounded-xl shadow-2xl p-8 mb-8">
            <div class="mb-6">
                <label for="location" class="block text-lg font-semibold text-blue-800 mb-2">Enter Location or Lat,Long (e.g., -31.9505,115.8605)</label>
                <div class="flex items-center mb-4">
                    <input type="text" id="location" placeholder="e.g., Perth, WA or -31.9505,115.8605" class="flex-grow p-3 border border-blue-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500">
                    <button onclick="getScoreFromInput()" class="ml-4 bg-blue-600 text-white p-3 rounded-lg hover:bg-blue-700 flex items-center">
                        <span id="buttonText">Get Nauti-Scores</span>
                        <div id="spinner" class="spinner ml-2"></div>
                    </button>
                </div>
                <div id="map"></div>
                <p class="text-sm text-gray-600 mt-2">Click or drag the marker to pinpoint a location. Current: <span id="coords">Lat: -31.9505, Lon: 115.8605</span></p>
            </div>
            <div id="error" class="hidden text-center text-red-600 mt-4 font-medium"></div>
        </div>
        <div id="result" class="hidden">
            <h2 class="text-2xl font-semibold text-center mb-6 text-white">7-Day Nauti-Score Forecast</h2>
            <div class="overflow-x-auto bg-white rounded-xl shadow-2xl">
                <table class="w-full table-auto">
                    <thead>
                        <tr class="bg-blue-700 text-white">
                            <th class="p-4 text-left">Date</th>
                            <th class="p-4 text-left">Nauti-Score</th>
                            <th class="p-4 text-left">Rain (mm)</th>
                            <th class="p-4 text-left">Wind (knots)</th>
                            <th class="p-4 text-left">Temp (°C)</th>
                            <th class="p-4 text-left">Est. Wave Ht (m)</th>
                            <th class="p-4 text-left">Conditions</th>
                        </tr>
                    </thead>
                    <tbody id="forecastTable" class="text-gray-800"></tbody>
                </table>
            </div>
        </div>
    </div>
    <footer class="bg-blue-900 text-white text-center p-4 mt-8">
        <p>© 2025 Nauti-Score. Set sail wisely! ⚓</p>
    </footer>
    <script>
        const API_KEY = '8b1ec085819de270f9a998cf32820758';
        let map, marker;

        // Initialize Leaflet map
        function initMap() {
            map = L.map('map').setView([-31.9505, 115.8605], 10); // Default: Perth, WA
            L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
                attribution: '© <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a>'
            }).addTo(map);
            marker = L.marker([-31.9505, 115.8605], { draggable: true }).addTo(map);
            marker.on('dragend', () => getScoreFromMap(marker.getLatLng()));
            map.on('click', (e) => {
                marker.setLatLng(e.latlng);
                getScoreFromMap(e.latlng);
            });
            // Fetch default data for Perth, WA
            getScore(-31.9505, 115.8605);
        }

        // Parse lat-long input
        function parseLatLong(input) {
            const latLongRegex = /^([-]?\d{1,3}\.\d+),([-]?\d{1,3}\.\d+)$/;
            const match = input.replace(/\s/g, '').match(latLongRegex);
            if (match) {
                const lat = parseFloat(match[1]);
                const lon = parseFloat(match[2]);
                if (lat >= -90 && lat <= 90 && lon >= -180 && lon <= 180) {
                    return { lat, lon };
                }
            }
            return null;
        }

        async function getCoordinates(location) {
            console.log(`Attempting to geocode location: ${location}`);
            const geocodingUrl = `http://api.openweathermap.org/geo/1.0/direct?q=${encodeURIComponent(location)}&limit=1&appid=${API_KEY}`;
            try {
                const response = await fetch(geocodingUrl);
                if (!response.ok) throw new Error(`Geocoding API error: ${response.status}`);
                const data = await response.json();
                if (data.length === 0) throw new Error('Location not found. Try a city name (e.g., "Perth, WA") or lat,long (e.g., "-31.9505,115.8605").');
                console.log(`Geocoding result:`, data);
                return { lat: data[0].lat, lon: data[0].lon };
            } catch (error) {
                throw new Error(error.message);
            }
        }

        async function getWeatherData(lat, lon) {
            const forecastUrl = `https://api.openweathermap.org/data/2.5/forecast?lat=${lat}&lon=${lon}&units=metric&appid=${API_KEY}`;
            try {
                const response = await fetch(forecastUrl);
                if (!response.ok) throw new Error(`Weather API error: ${response.status}`);
                const data = await response.json();
                return aggregateDailyData(data.list);
            } catch (error) {
                throw new Error('Error fetching weather data: ' + error.message);
            }
        }

        function aggregateDailyData(forecastList) {
            const dailyData = {};
            forecastList.forEach(item => {
                const date = new Date(item.dt * 1000).toISOString().split('T')[0];
                if (!dailyData[date]) {
                    dailyData[date] = {
                        temp: [],
                        wind_speed: [],
                        visibility: [],
                        rain: 0,
                        weather: item.weather[0].main
                    };
                }
                dailyData[date].temp.push(item.main.temp);
                dailyData[date].wind_speed.push(item.wind.speed * 1.94384); // Convert m/s to knots
                dailyData[date].visibility.push(item.visibility || 10000);
                if (item.rain && item.rain['3h']) dailyData[date].rain += item.rain['3h'];
                if (item.weather[0].main.toLowerCase().includes('rain')) dailyData[date].weather = 'Rain';
            });

            const dailyArray = Object.keys(dailyData).map(date => ({
                dt: new Date(date).getTime() / 1000,
                main: { temp: average(dailyData[date].temp) },
                wind: { speed: average(dailyData[date].wind_speed) },
                visibility: average(dailyData[date].visibility),
                rain: dailyData[date].rain,
                weather: [{ main: dailyData[date].weather }]
            }));

            const today = new Date();
            const result = [];
            for (let i = 0; i < 7; i++) {
                const date = new Date(today);
                date.setDate(today.getDate() + i);
                const dateStr = date.toISOString().split('T')[0];
                const dayData = dailyArray.find(d => new Date(d.dt * 1000).toISOString().split('T')[0] === dateStr);
                result.push(dayData || {
                    dt: date.getTime() / 1000,
                    main: { temp: 0 },
                    wind: { speed: 0 },
                    visibility: 0,
                    rain: 0,
                    weather: [{ main: 'N/A' }],
                    unavailable: true
                });
            }

            return result;

            function average(arr) {
                return arr.length ? arr.reduce((a, b) => a + b, 0) / arr.length : 0;
            }
        }

        function calculateNautiScore(weather) {
            if (weather.unavailable) {
                return { score: 'N/A', message: 'Data unavailable (API limited to 5 days)', waveHeight: 0 };
            }

            let score = 100;

            // Wind speed (knots): >13 knots is bad, <4.3 is great
            const windSpeed = weather.wind.speed;
            if (windSpeed > 13) score -= (windSpeed - 13) * 3.46; // Adjusted for knots (3 * 15/13)
            else if (windSpeed < 4.3) score += 5;

            // Wave height (meters, proxy for swell): >1.22 m is bad
            const waveHeight = 0.00267 * windSpeed * windSpeed; // In meters, using knots
            if (waveHeight > 1.22) score -= (waveHeight - 1.22) * 16.4; // Adjusted for meters (5 * 4/1.22)

            // Precipitation: Heavy penalty for any rain
            const rain = weather.rain || (weather.weather[0].main.toLowerCase().includes('rain') ? 1 : 0);
            if (rain > 0) score -= 30 + Math.min(rain * 10, 20);

            // Temperature (°C): 16–29°C is ideal
            const temp = weather.main.temp;
            if (temp < 16) score -= (16 - temp) * 0.9; // Adjusted for °C (0.5 * 9/5)
            else if (temp > 29) score -= (temp - 29) * 0.9;

            // Visibility (meters, convert to km): <8 km is bad
            const visibilityKm = (weather.visibility || 10000) / 1000;
            if (visibilityKm < 8) score -= (8 - visibilityKm) * 1.24; // Adjusted for km (2 * 5/8)

            // Clamp score between 1 and 100
            score = Math.max(1, Math.min(100, Math.round(score)));

            // Generate message
            let message = '';
            if (score >= 90) message = "Perfect day! Set sail! ☀️";
            else if (score >= 70) message = "Smooth seas! Good for boating. 🚤";
            else if (score >= 50) message = "Choppy, sail with caution. 🌊";
            else if (score >= 30) message = "Sketchy, brave sailors only. ⚓";
            else message = "Stay docked, sea’s angry! 🌪️";

            return { score, message, waveHeight };
        }

        async function getScore(lat, lon) {
            const resultDiv = document.getElementById('result');
            const errorDiv = document.getElementById('error');
            const forecastTable = document.getElementById('forecastTable');
            const spinner = document.getElementById('spinner');
            const buttonText = document.getElementById('buttonText');

            // Reset UI
            resultDiv.classList.add('hidden');
            errorDiv.classList.add('hidden');
            errorDiv.textContent = '';
            forecastTable.innerHTML = '';
            buttonText.textContent = 'Fetching...';
            spinner.style.display = 'block';

            try {
                // Get 5-day forecast
                const forecast = await getWeatherData(lat, lon);

                // Calculate scores and populate table
                forecast.forEach((day, index) => {
                    const { score, message, waveHeight } = calculateNautiScore(day);
                    const date = new Date(day.dt * 1000);
                    const isWeekend = date.getDay() === 0 || date.getDay() === 6;
                    const dateStr = date.toLocaleDateString('en-US', {
                        weekday: 'short',
                        month: 'short',
                        day: 'numeric'
                    });
                    if (isWeekend) {
                        console.log(`Weather for ${dateStr}:`, {
                            rain: day.rain,
                            windSpeed: day.wind.speed,
                            temp: day.main.temp,
                            waveHeight,
                            visibility: day.visibility / 1000,
                            weather: day.weather[0].main,
                            score
                        });
                    }
                    const row = `
                        <tr class="${index % 2 === 0 ? 'bg-blue-50' : 'bg-white'} border-b">
                            <td class="p-4 font-medium">${dateStr}</td>
                            <td class="p-4">${score}</td>
                            <td class="p-4">${day.unavailable ? 'N/A' : day.rain.toFixed(2)}</td>
                            <td class="p-4">${day.unavailable ? 'N/A' : day.wind.speed.toFixed(1)}</td>
                            <td class="p-4">${day.unavailable ? 'N/A' : day.main.temp.toFixed(1)}</td>
                            <td class="p-4">${day.unavailable ? 'N/A' : waveHeight.toFixed(2)}</td>
                            <td class="p-4">${message}</td>
                        </tr>
                    `;
                    forecastTable.innerHTML += row;
                });

                // Show table
                resultDiv.classList.remove('hidden');
            } catch (error) {
                errorDiv.textContent = error.message;
                errorDiv.classList.remove('hidden');
            } finally {
                buttonText.textContent = 'Get Nauti-Scores';
                spinner.style.display = 'none';
            }
        }

        async function getScoreFromInput() {
            const location = document.getElementById('location').value.trim();
            const errorDiv = document.getElementById('error');
            const buttonText = document.getElementById('buttonText');
            const spinner = document.getElementById('spinner');

            // Reset error
            errorDiv.classList.add('hidden');
            errorDiv.textContent = '';
            buttonText.textContent = 'Fetching...';
            spinner.style.display = 'block';

            try {
                let lat, lon;
                // Check if input is lat,long
                const coords = parseLatLong(location);
                if (coords) {
                    ({ lat, lon } = coords);
                    map.setView([lat, lon], 10);
                    marker.setLatLng([lat, lon]);
                    document.getElementById('coords').textContent = `Lat: ${lat.toFixed(4)}, Lon: ${lon.toFixed(4)}`;
                } else if (location) {
                    // Try geocoding for city/place
                    ({ lat, lon } = await getCoordinates(location));
                    map.setView([lat, lon], 10);
                    marker.setLatLng([lat, lon]);
                    document.getElementById('coords').textContent = `Lat: ${lat.toFixed(4)}, Lon: ${lon.toFixed(4)}`;
                } else {
                    // Fallback to map marker's coordinates
                    const latlng = marker.getLatLng();
                    lat = latlng.lat;
                    lon = latlng.lng;
                    document.getElementById('location').value = `${lat.toFixed(4)},${lon.toFixed(4)}`;
                    document.getElementById('coords').textContent = `Lat: ${lat.toFixed(4)}, Lon: ${lon.toFixed(4)}`;
                }
                await getScore(lat, lon);
            } catch (error) {
                errorDiv.textContent = error.message;
                errorDiv.classList.remove('hidden');
                buttonText.textContent = 'Get Nauti-Scores';
                spinner.style.display = 'none';
            }
        }

        async function getScoreFromMap(latlng) {
            document.getElementById('location').value = `${latlng.lat.toFixed(4)},${latlng.lng.toFixed(4)}`;
            document.getElementById('coords').textContent = `Lat: ${latlng.lat.toFixed(4)}, Lon: ${latlng.lng.toFixed(4)}`;
            await getScore(latlng.lat, latlng.lng);
        }

        // Initialize map on page load
        window.onload = initMap;
    </script>
</body>
</html># Nauti-score
