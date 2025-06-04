<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8" />
  <title>Carte 5 Points - Itinéraire</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css" />
  <link rel="stylesheet" href="https://unpkg.com/leaflet-routing-machine/dist/leaflet-routing-machine.css" />
  <style>
    #map { height: 100vh; }
    #controls {
      position: absolute;
      top: 10px;
      left: 10px;
      z-index: 1000;
      background: white;
      padding: 10px;
      border-radius: 8px;
      box-shadow: 0 0 8px rgba(0,0,0,0.3);
      max-width: 300px;
    }
    #results {
      margin-top: 10px;
      font-size: 14px;
    }
    li {
      margin-bottom: 6px;
    }
  </style>
</head>
<body>
  <div id="controls">
    <label for="destination">Destination :</label>
    <input type="text" id="destination" placeholder="Ex: 10 rue de Rivoli, Paris">
    <button onclick="calculateAllRoutes()">Comparer itinéraires</button>
    <div id="results"></div>
  </div>
  <div id="map"></div>

  <script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>
  <script src="https://unpkg.com/leaflet-routing-machine/dist/leaflet-routing-machine.js"></script>
  <script>
    const map = L.map('map').setView([48.845, 2.36], 12);

    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      maxZoom: 19,
      attribution: '&copy; OpenStreetMap contributors'
    }).addTo(map);

    const points = [
      { name: "Bercy", coords: [48.8324, 2.3874] },
      { name: "Gare de Lyon", coords: [48.8412, 2.3723] },
      { name: "Place d'Italie", coords: [48.8362, 2.3613] },
      { name: "Boulogne-Billancourt", coords: [48.8297, 2.2547] },
      { name: "Neuilly Saint-Jean-Baptiste", coords: [48.8847, 2.2669] }
    ];

    let routeLines = [];

    points.forEach(p => {
      L.marker(p.coords).addTo(map).bindPopup(p.name);
    });

    function clearRoutes() {
      routeLines.forEach(line => map.removeControl(line));
      routeLines = [];
    }

    function calculateAllRoutes() {
      const destination = document.getElementById('destination').value;
      const resultsDiv = document.getElementById('results');
      resultsDiv.innerHTML = "Calcul en cours...";

      clearRoutes();

      // Geocode destination
      fetch(`https://nominatim.openstreetmap.org/search?format=json&q=${encodeURIComponent(destination)}`)
        .then(res => res.json())
        .then(data => {
          if (data.length === 0) {
            resultsDiv.innerHTML = "Adresse introuvable.";
            return;
          }

          const destCoords = L.latLng(parseFloat(data[0].lat), parseFloat(data[0].lon));

          const routePromises = points.map(p => {
            return new Promise((resolve, reject) => {
              const router = L.Routing.control({
                waypoints: [
                  L.latLng(p.coords), destCoords
                ],
                routeWhileDragging: false,
                draggableWaypoints: false,
                addWaypoints: false,
                createMarker: () => null,
                show: false // Disable the itinerary instructions panel
              });

              router.on('routesfound', function(e) {
                const route = e.routes[0];
                resolve({
                  name: p.name,
                  distance: route.summary.totalDistance / 1000, // km
                  duration: route.summary.totalTime / 60, // minutes
                  line: router
                });
              });

              router.on('routingerror', function() {
                reject();
              });

              router.addTo(map);
              routeLines.push(router);
            });
          });

          Promise.all(routePromises)
            .then(results => {
              results.sort((a, b) => a.distance - b.distance);
              resultsDiv.innerHTML = '<b>Classement des trajets :</b><ul>' +
                results.map(r => `<li><b>${r.name}</b> → ${r.distance.toFixed(2)} km / ${r.duration.toFixed(1)} min</li>`).join('') +
                '</ul>';
            })
            .catch(err => {
              resultsDiv.innerHTML = "Erreur de calcul des itinéraires.";
            });
        })
        .catch(err => {
          resultsDiv.innerHTML = "Erreur lors de la géolocalisation de l'adresse.";
        });
    }
  </script>
</body>
</html>
