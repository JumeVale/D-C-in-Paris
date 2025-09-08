<!DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="UTF-8" />
  <title>Classement des trajets</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css" />
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
    "Neuilly Saint-Jean-Baptiste": [48.8847, 2.2669],
    "La Défense": [48.8919, 2.2401],
    "D&C Wurtz": [48.8220, 2.3488]
  };

  function getBoostMessage(distanceKm) {
    if (distanceKm < 6) return "No boost required";
    if (distanceKm < 10) return "Apply €3 boost";
    if (distanceKm < 15) return "Apply €6 boost";
    return "Apply €6 boost and inform OPS team";
  }

  // Utilitaires robustes
  async function fetchJSON(url, { timeoutMs = 12000 } = {}) {
    const ctrl = new AbortController();
    const t = setTimeout(() => ctrl.abort(), timeoutMs);
    try {
      const res = await fetch(url, { signal: ctrl.signal });
      if (!res.ok) {
        // Message clair selon le code
        throw new Error(`HTTP ${res.status}`);
      }
      // Tente de parser JSON (si HTML renvoyé par erreur, ça throw ici)
      return await res.json();
    } finally {
      clearTimeout(t);
    }
  }

  function drawRoute(geojson) {
    try {
      L.geoJSON(geojson, { color: 'blue' }).addTo(map);
    } catch (e) {
      // Ignore un tracé mal formé, mais ne casse pas l'appli
      console.warn('Impossible d’afficher la géométrie', e);
    }
  }

  async function routeOSRM(fromLat, fromLon, toLat, toLon) {
    // D’abord avec alternatives; si ça échoue, on retente plus simple
    const base = `https://router.project-osrm.org/route/v1/driving/${fromLon},${fromLat};${toLon},${toLat}`;
    const params1 = `?overview=full&geometries=geojson&alternatives=true`;
    const params2 = `?overview=simplified&geometries=geojson&alternatives=false`;

    // Essai 1
    try {
      const data = await fetchJSON(base + params1);
      if (!data || !Array.isArray(data.routes) || data.routes.length === 0) {
        throw new Error('No route');
      }
      return data.routes;
    } catch (e1) {
      // Essai 2 (fallback)
      const data = await fetchJSON(base + params2);
      if (!data || !Array.isArray(data.routes) || data.routes.length === 0) {
        throw new Error('No route');
      }
      return data.routes;
    }
  }

  async function calculateAllRoutes() {
    const destinationInput = document.getElementById('destination').value.trim();
    const resultsDiv = document.getElementById('results');
    resultsDiv.innerHTML = "Calcul en cours...";

    // Nettoyage des polylignes/markers précédents mais on garde les tuiles
    map.eachLayer(layer => {
      if (layer instanceof L.Polyline || (layer instanceof L.Marker && !layer._url)) {
        map.removeLayer(layer);
      }
    });

    // Géocodage (Nominatim : ajouter &limit=1 &accept-language=fr)
    let destLat, destLon;
    try {
      if (!destinationInput) throw new Error('empty');
      const geoUrl = `https://nominatim.openstreetmap.org/search?format=json&limit=1&accept-language=fr&q=${encodeURIComponent(destinationInput)}`;
      const geoData = await fetchJSON(geoUrl, { timeoutMs: 15000 });
      if (!Array.isArray(geoData) || geoData.length === 0) throw new Error('notfound');
      destLat = parseFloat(geoData[0].lat);
      destLon = parseFloat(geoData[0].lon);
    } catch (e) {
      resultsDiv.innerHTML = "Adresse introuvable ou service de géocodage indisponible.";
      console.error('Erreur Nominatim:', e);
      return;
    }

    const destCoords = [destLat, destLon];
    L.marker(destCoords).addTo(map).bindPopup("Destination").openPopup();

    const results = [];

    // IMPORTANT : sérialiser pour éviter le ratelimiting OSRM
    for (const [name, coords] of Object.entries(points)) {
      try {
        const routes = await routeOSRM(coords[0], coords[1], destLat, destLon);
        // Choisir la meilleure route (durée mini)
        const bestRoute = routes.reduce((min, r) => (r.duration < min.duration ? r : min), routes[0]);
        const distanceKm = bestRoute.distance / 1000;
        const durationMin = bestRoute.duration / 60;
        const boost = getBoostMessage(distanceKm);

        // Tracé
        if (bestRoute.geometry) {
          drawRoute(bestRoute.geometry);
        }

        // Marker origine
        L.marker(coords).addTo(map).bindPopup(name);

        results.push({ name, distanceKm, durationMin, boost, ok: true });
      } catch (e) {
        console.warn(`Erreur OSRM pour ${name}:`, e);
        results.push({ name, ok: false, reason: e.message || 'Erreur inconnue' });
      }

      // Petite pause (200ms) pour être gentil avec le serveur public
      await new Promise(r => setTimeout(r, 200));
    }

    // Affichage
    const valid = results.filter(r => r.ok).sort((a, b) => a.distanceKm - b.distanceKm);
    let output = '<b>Classement par distance (km)</b>';
    if (valid.length) {
      output += '<ul>' + valid.map(r =>
        `<li><b>${r.name}</b>: ${r.distanceKm.toFixed(2)} km - ${r.durationMin.toFixed(1)} min<br>${r.boost}</li>`
      ).join('') + '</ul>';
    } else {
      output += '<p>Aucun itinéraire disponible pour le moment.</p>';
    }

    const errors = results.filter(r => !r.ok);
    if (errors.length) {
      output += '<b>Erreurs :</b><ul>' + errors.map(r =>
        `<li><b>${r.name}</b> : ${r.reason}</li>`
      ).join('') + '</ul>';
    }

    resultsDiv.innerHTML = output;
  }

  // Rendre la fonction globale pour le bouton
  window.calculateAllRoutes = calculateAllRoutes;
</script>
</body>
</html>
