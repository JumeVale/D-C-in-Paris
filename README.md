<!DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="UTF-8" />
  <title>Classement des trajets</title>
</head>
<body>
  <label for="destination">Destination :</label>
  <input type="text" id="destination" placeholder="10 rue de Rivoli, Paris" />
  <button onclick="calculateAllRoutes()">Calculer</button>
  <div id="results"></div>

  <script>
    const points = {
      "Bercy": [48.8324, 2.3874],
      "Gare de Lyon": [48.8412, 2.3723],
      "Place d'Italie": [48.8362, 2.3613],
      "Boulogne-Billancourt": [48.8297, 2.2547],
      "Neuilly Saint-Jean-Baptiste": [48.8847, 2.2669],
      "La Défense": [48.8919, 2.2401],
      "D&C Wurtz": [48.8220, 2.3488]
    };

    function getBoostMessage(d) {
      if (d < 6) return "No boost required";
      if (d < 10) return "Apply €3 boost";
      if (d < 15) return "Apply €6 boost";
      return "Apply €6 boost and inform OPS team";
    }

    async function calculateAllRoutes() {
      const dest = document.getElementById('destination').value;
      const resultsDiv = document.getElementById('results');
      resultsDiv.textContent = "Calcul en cours...";

      try {
        const geo = await (await fetch(`https://nominatim.openstreetmap.org/search?format=json&q=${encodeURIComponent(dest)}`)).json();
        if (!geo.length) return resultsDiv.textContent = "Adresse introuvable.";

        const [lat, lon] = [geo[0].lat, geo[0].lon];
        const coordsList = Object.values(points).map(p => `${p[1]},${p[0]}`);
        const urlTable = `https://router.project-osrm.org/table/v1/driving/${coordsList.join(';')};${lon},${lat}?annotations=duration,distance`;

        const tableRes = await fetch(urlTable);
        const tableData = await tableRes.json();

        if (tableData.code === "Ok") {
          const durations = tableData.durations.map(row => row[row.length - 1]); // last column = destination
          const distances = tableData.distances.map(row => row[row.length - 1]);

          const results = Object.keys(points).map((name, i) => ({
            name,
            distanceKm: distances[i] / 1000,
            durationMin: durations[i] / 60,
            boost: getBoostMessage(distances[i] / 1000)
          })).sort((a, b) => a.distanceKm - b.distanceKm);

          resultsDiv.innerHTML = "<ul>" +
            results.map(r => `<li>${r.name}: ${r.distanceKm.toFixed(2)} km, ${r.durationMin.toFixed(1)} min — ${r.boost}</li>`).join('') +
            "</ul>";

        } else {
          throw new Error("Table API fallback");
        }
      } catch (err) {
        resultsDiv.innerHTML = "Erreur ou OSRM indisponible.";
        console.error(err);
      }
    }
  </script>
</body>
</html>
