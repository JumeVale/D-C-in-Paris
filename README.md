<!DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="UTF-8" />
  <title>Classement des trajets</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <link 
    rel="stylesheet" 
    href="https://unpkg.com/leaflet/dist/leaflet.css" 
  />
  <style>
    html, body {
      height: 100%;
      margin: 0;
      padding: 0;
    }
    #map {
      position: absolute;
      top: 0;
      left: 0;
      height: 100%;
      width: 100%;
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
      max-height: 300px;
      overflow-y: auto;
    }
  </style>
</head>
<body>
  <div id="controls">
    <label for="destination">Destination :</label>
    <input type="text" id="destination" placeholder="Ex: 10 rue de Rivoli, Paris" />
    <button onclick="calculateAllRoutes()">Calculer tous les trajets</button>
    <div id="results"></div>
  </div>
  <div id="map"></div>

  <script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>
  <script>
    // 🔑 Clé OpenRouteService
    const API_KEY = "eyJvcmciOiI1YjNjZTM1OTc4NTExMTAwMDFjZjYyNDgiLCJpZCI6IjYzNDI2YzNkNmFjYjQ2ZTJiMWQ1NjA5ZmE3YWQ3OWU1IiwiaCI6Im11cm11cjY0In0=";

    // Initialisation de la carte
    const map = L.map('map').setView([48.845, 2.36], 12);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      maxZoom: 19,
      attribution: '&copy; OpenStreetMap contributors'
    }).addTo(map);

    // Points de départ
    const points = {
      "Bercy": [48.8324, 2.3874],
      "Gare de Lyon": [48.8412, 2.3723],
      "Place d'Italie": [48.8362, 2.3613],
      "Boulogne-Billancourt": [48.8297, 2.2547],
      "Neuilly Saint-Jean-Baptiste": [48.8847, 2.2669],
      "La Défense": [48.8919, 2.2401],
      "D&C Wurtz": [48.8220, 2.3488]
    };

    // Fonction boost
    function getBoostMessage(distanceKm) {
      if (distanceKm < 6) return "No boost required";
      if (distanceKm < 10) return "Apply €3 boost";
      if (distanceKm < 15) return "Apply €6 boost";
      return "Apply €6 boost and inform OPS team";
    }

    // Requête OpenRouteService
    async function fetchRouteORS(startCoords, destCoords) {
      const url = "https://api.openrouteservice.org/v2/directions/driving-car";
      const body = {
        coordinates: [
          [startCoords[1], startCoords[0]], // ORS attend [lon, lat]
          [destCoords[1], destCoords[0]]
        ]
      };

      const res = await fetch(url, {
        method: "POST",
        headers: {
          "Authorization": API_KEY,
          "Content-Type": "application/json"
        },
        body: JSON.stringify(body)
      });

      if (!res.ok) throw new Error("ORS request failed");
      return res.json();
    }

    // Calcul des trajets
    async function calculateAllRoutes() {
      const destinationInput = document.getElementById('destination').value;
      const resultsDiv = document.getElementById('results');
      resultsDiv.innerHTML = "Calcul en cours...";

      // On efface anciens tracés et marqueurs
      map.eachLayer(layer => {
        if (layer instanceof L.Polyline || layer instanceof L.Marker) {
          if (!layer._url) map.removeLayer(layer); // garde le fond de carte
        }
      });

      try {
        // Géocodage avec Nominatim
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

        // Calcul pour chaque point
        const results = await Promise.all(Object.entries(points).map(async ([name, coords]) => {
          try {
            const data = await fetchRouteORS(coords, destCoords);

            const summary = data.features[0].properties.summary;
            const distanceKm = summary.distance / 1000;
            const durationMin = summary.duration / 60;
            const boost = getBoostMessage(distanceKm);

            // Ajout sur la carte
            const geometry = data.features[0].geometry;
            L.geoJSON(geometry, { color: 'blue' }).addTo(map);
            L.marker(coords).addTo(map).bindPopup(name);

            return { name, distanceKm, durationMin, boost };
          } catch (err) {
            console.error("Erreur ORS pour " + name, err);
            return { name, error: true };
          }
        }));

        // Classement
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
