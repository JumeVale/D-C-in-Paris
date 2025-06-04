<html>
<head>
  <meta charset="utf-8" />
  <title>Classement des trajets par distance</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css" />
  <style>
    html, body {
      height: 100%;
      margin: 0;
    }
    #map {
      height: 100%;
    }
    #controls {
      position: absolute;
      top: 10px;
      left: 10px;
      z-index: 1000;
      background: white;
      padding: 10px;
      border-radius: 8px;
      box-shadow: 0 0 8px rgba(0,0,0,0.3);
      max-width: 350px;
    }
    #results {
      margin-top: 10px;
      font-size: 14px;
    }
  </style>
</head>
<body>
  <div id="controls">
    <label for="destination">Destination :</label>
    <input type="text" id="destination" placeholder="Ex: 10 rue de Rivoli, Paris">
    <button onclick="calculateAllRoutes()">Calculer tous les trajets</button>
    <div id="results"></div>
  </div>
  <div id="map"></div>

  <script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>
  <script>
    const map = L.map('map').setView([48.845, 2.36], 12);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      maxZoom: 19,
      attribution: '&copy; OpenStreetMap contributors'
    }).addTo(map);

    const points = {
      "Bercy": [48.8324, 2.3874],
      "Gare de Lyon": [48.8412, 2.3723],
      "Place d'Italie": [48.8362, 2.3613],
      "Boulogne-Billancourt": [48.8297, 2.2547],
      "Neuilly Saint-Jean-Baptiste": [48.8847, 2.2669]
    };

    function getBoostMessage(distanceKm) {
      if (distanceKm < 6) return "No boost required";
      if (distanceKm < 10) return "Apply €3 boost";
      if (distanceKm < 15) return "Apply €6 boost";
      return "Apply €6 boost and inform OPS team";
    }

    async function calculateAllRoutes() {
      const destinationInput = document.getElementById('destination').value;
      const resultsDiv = document.getElementById('results');
      resultsDiv.innerHTML = "Calcul en cours...";

      try {
        const geoRes = await fetch(`https://nominatim.openstreetmap.org/search?format=json&q=${encodeURIComponent(destinationInput)}`);
        const geoData = await geoRes.json();

        if (!geoData.length) {
          resultsDiv.innerHTML = "Adresse introuvable.";
          return;
        }

        const destLat = parseFloat(geoData[0].lat);
        const destLon = parseFloat(geoData[0].lon);
        const destCoords = [destLat, destLon];
        L.marker(destCoords).addTo(map).bindPopup("Destination").openPopup();

        const results = await Promise.all(Object.entries(points).map(async ([name, coords]) => {
          const url = `https://router.project-osrm.org/route/v1/driving/${coords[1]},${coords[0]};${destLon},${destLat}?overview=full&geometries=geojson`;
          try {
            const res = await fetch(url);
            const data = await res.json();
            if (!data.routes || !data.routes.length) throw new Error('No route');

            const r = data.routes[0];
            const distanceKm = r.distance / 1000;
            const durationMin = r.duration / 60;
            const boost = getBoostMessage(distanceKm);

            L.geoJSON(r.geometry, { color: 'blue' }).addTo(map);
            L.marker(coords).addTo(map).bindPopup(name);

            return { name, distanceKm, durationMin, boost };
          } catch {
            return { name, error: true };
          }
        }));

        const valid = results.filter(r => !r.error);
        valid.sort((a, b) => a.distanceKm - b.distanceKm);

        let output = '<b>Classement par distance (km)</b><ul>' +
          valid.map(r => `<li><b>${r.name}</b>: ${r.distanceKm.toFixed(2)} km - ${r.durationMin.toFixed(1)} min<br>${r.boost}</li>`).join('') + '</ul>';

        const errors = results.filter(r => r.error);
        if (errors.length) {
          output += '<b>Erreurs :</b><ul>' +
            errors.map(r => `<li><b>${r.name}</b>: itinéraire non disponible</li>`).join('') + '</ul>';
        }

        resultsDiv.innerHTML = output;

      } catch (err) {
        resultsDiv.innerHTML = "Erreur lors du calcul.";
        console.error(err);
      }
    }
  </script>
</body>
</html>
