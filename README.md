# Interactive-World-Map[index.html](https://github.com/user-attachments/files/24380745/index.html)
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Global Explorer - UN Edition</title>
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif; }
        
        body, html { height: 100%; width: 100%; overflow: hidden; background: #eef2f3; }

        /* Warning Banner */
        #warning-banner {
            position: fixed;
            top: 0;
            width: 100%;
            background: #fff3cd;
            color: #856404;
            padding: 12px 15px;
            font-size: 11px;
            text-align: center;
            z-index: 2000;
            border-bottom: 1px solid #ffeeba;
            line-height: 1.4;
            box-shadow: 0 2px 4px rgba(0,0,0,0.05);
        }

        #map { height: 100vh; width: 100vw; z-index: 1; }

        /* Bottom Sheet for Mobile / Sidebar for Desktop */
        #details-panel {
            position: fixed;
            bottom: 0;
            left: 0;
            width: 100%;
            background: white;
            padding: 20px;
            border-radius: 20px 20px 0 0;
            box-shadow: 0 -5px 25px rgba(0,0,0,0.15);
            z-index: 1001;
            transform: translateY(100%);
            transition: transform 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.275);
            max-height: 60vh;
            overflow-y: auto;
        }

        #details-panel.active { transform: translateY(0); }

        .header-row { display: flex; justify-content: space-between; align-items: flex-start; margin-bottom: 15px; }
        .flag-img { width: 70px; border-radius: 4px; border: 1px solid #eee; box-shadow: 0 2px 5px rgba(0,0,0,0.1); }
        .close-btn { font-size: 32px; color: #ccc; cursor: pointer; line-height: 0.8; }

        .stat-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; margin-top: 10px; }
        .stat-item { background: #f8f9fa; padding: 12px; border-radius: 10px; border: 1px solid #f0f0f0; }
        .stat-item label { display: block; font-size: 10px; text-transform: uppercase; color: #888; letter-spacing: 0.6px; margin-bottom: 4px; }
        .stat-item span { display: block; font-size: 14px; font-weight: 600; color: #222; }
        
        /* Language box spans both columns if the text is long */
        .full-width { grid-column: span 2; }

        @media (min-width: 768px) {
            #details-panel {
                top: 80px; right: 20px; left: auto; bottom: auto;
                width: 360px; border-radius: 15px; transform: translateX(120%);
                max-height: 80vh;
            }
            #details-panel.active { transform: translateX(0); }
            #warning-banner { font-size: 13px; padding: 15px; }
        }
    </style>
</head>
<body>

    <div id="warning-banner">
        ⚠️ <strong>WARNING:</strong> This website uses official world map according to UN. It may or may not portray a Country's official boundaries correctly.
    </div>

    <div id="details-panel">
        <div class="header-row">
            <div id="title-area">
                <h2 id="country-name" style="font-size: 1.6rem; color: #2c3e50;">Country Name</h2>
                <p id="region" style="font-size: 13px; color: #7f8c8d;"></p>
            </div>
            <div style="display: flex; align-items: center; gap: 15px;">
                <img id="flag-icon" class="flag-img" src="" alt="">
                <span class="close-btn" onclick="togglePanel(false)">&times;</span>
            </div>
        </div>

        <div class="stat-grid">
            <div class="stat-item">
                <label>Capital</label>
                <span id="capital">-</span>
            </div>
            <div class="stat-item">
                <label>Population</label>
                <span id="population">-</span>
            </div>
            <div class="stat-item">
                <label>Currency</label>
                <span id="currency">-</span>
            </div>
            <div class="stat-item">
                <label>Official Languages</label>
                <span id="languages">-</span>
            </div>
        </div>
    </div>

    <div id="map"></div>

    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
    <script>
        // Initialize Map
        const map = L.map('map', { center: [20, 10], zoom: 2, zoomControl: false });
        
        // Position zoom controls for better thumb reach on mobile
        L.control.zoom({ position: 'bottomright' }).addTo(map);

        // Clean background tiles
        L.tileLayer('https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png', {
            attribution: '&copy; CartoDB | UN World Map'
        }).addTo(map);

        let geojsonLayer;

        async function init() {
            try {
                // Fetch standard UN-style GeoJSON
                const response = await fetch('https://raw.githubusercontent.com/datasets/geo-countries/master/data/countries.geojson');
                const data = await response.json();

                geojsonLayer = L.geoJSON(data, {
                    style: { color: '#3498db', weight: 1.5, fillOpacity: 0.15, fillColor: '#3498db' },
                    onEachFeature: (feature, layer) => {
                        layer.on('click', (e) => {
                            const name = feature.properties.ADMIN || feature.properties.name;
                            fetchData(name);
                            // Padding ensures the country isn't hidden by the UI panels
                            map.fitBounds(e.target.getBounds(), { padding: [60, 60] });
                        });
                        
                        // Visual hover feedback for desktop
                        layer.on('mouseover', () => layer.setStyle({ fillOpacity: 0.4 }));
                        layer.on('mouseout', () => layer.setStyle({ fillOpacity: 0.15 }));
                    }
                }).addTo(map);
            } catch (err) {
                console.error("Failed to load map boundaries:", err);
            }
        }

        async function fetchData(name) {
            togglePanel(true);
            const nameEl = document.getElementById('country-name');
            nameEl.innerText = name;
            
            try {
                const res = await fetch(`https://restcountries.com/v3.1/name/${name}?fullText=true`);
                if (!res.ok) throw new Error("Not found");
                
                const data = await res.json();
                const c = data[0];

                document.getElementById('flag-icon').src = c.flags.png;
                document.getElementById('region').innerText = c.subregion || c.region;
                document.getElementById('capital').innerText = c.capital ? c.capital[0] : 'N/A';
                document.getElementById('population').innerText = c.population.toLocaleString();
                
                // Handle Currency
                const curKey = Object.keys(c.currencies)[0];
                const currency = c.currencies[curKey];
                document.getElementById('currency').innerText = `${currency.name} (${currency.symbol || ''})`;
                
                // Handle Languages
                const langs = Object.values(c.languages).join(", ");
                document.getElementById('languages').innerText = langs;

            } catch (err) {
                document.getElementById('capital').innerText = "Data error";
                console.error(err);
            }
        }

        function togglePanel(show) {
            document.getElementById('details-panel').classList.toggle('active', show);
        }

        init();
    </script>
</body>
</html>
