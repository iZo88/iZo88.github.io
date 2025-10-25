
<html lang="el">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Arduino Sensors & Pump Control</title>
  <script src="https://cdn.jsdelivr.net/npm/chart.js">
// --- Αποθήκευση επιλογών στο LocalStorage ---
function saveSettings() {
  const pumpMode = document.getElementById("pumpMode").value;
  const pumpControl = document.querySelector("input[name='pumpControl']:checked").value;
  const humidityLimit = document.getElementById("humidityLimit").value;

  localStorage.setItem("pumpMode", pumpMode);
  localStorage.setItem("pumpControl", pumpControl);
  localStorage.setItem("humidityLimit", humidityLimit);
}

// --- Επαναφορά επιλογών κατά το άνοιγμα της σελίδας ---
function loadSettings() {
  const pumpMode = localStorage.getItem("pumpMode");
  const pumpControl = localStorage.getItem("pumpControl");
  const humidityLimit = localStorage.getItem("humidityLimit");

  if (pumpMode !== null) {
    document.getElementById("pumpMode").value = pumpMode;
  }
  if (pumpControl !== null) {
    document.querySelector(`input[name='pumpControl'][value='${pumpControl}']`).checked = true;
  }
  if (humidityLimit !== null) {
    document.getElementById("humidityLimit").value = humidityLimit;
  }
}

// --- Συνδυασμός με αποστολή εντολής ---
const originalSendControl = sendControl;
sendControl = async function() {
  const pumpMode = document.getElementById("pumpMode").value;
  const pumpControl = document.querySelector("input[name='pumpControl']:checked").value;
  const humidityLimit = document.getElementById("humidityLimit").value;

  const url = `https://api.thingspeak.com/update?api_key=${writeAPIKey}&field5=${pumpMode}&field6=${pumpControl}&field7=${humidityLimit}`;
  await fetch(url);
  alert("Εντολή στάλθηκε στο Arduino!");

  // Αποθήκευση επιλογών
  saveSettings();
};

// --- Φόρτωση ρυθμίσεων μόλις ανοίξει η σελίδα ---
window.addEventListener("load", loadSettings);

</script>
  <style>
    body {
      font-family: Arial, sans-serif;
      text-align: center;
      background: #f4f4f4;
      margin: 20px;
    }
    h1 {
      margin-bottom: 30px;
    }
    .chart-container, .control-container {
      width: 80%;
      margin: 20px auto;
      background: #fff;
      padding: 20px;
      border-radius: 12px;
      box-shadow: 0 4px 10px rgba(0,0,0,0.1);
    }
    canvas {
      max-width: 100%;
    }
    label {
      font-weight: bold;
    }
    .control-group {
      margin: 15px 0;
    }
  </style>
</head>
<body>
  <h1>Arduino Sensors Dashboard</h1>

  <!-- Controls -->
  <div class="control-container">
    <h2>Έλεγχος Αντλίας</h2>

    <div class="control-group">
      <label for="pumpMode">Αντλία - Κατάσταση:</label>
      <select id="pumpMode">
        <option value="1">Αυτόματη</option>
        <option value="0">Χειροκίνητη</option>
      </select>
    </div>

    <div class="control-group">
      <label>Αντλία - ON/OFF:</label><br>
      <input type="radio" name="pumpControl" value="1"> ON
      <input type="radio" name="pumpControl" value="0" checked> OFF
    </div>

    <div class="control-group">
      <label for="humidityLimit">Όριο Υγρασία Εδάφους:</label>
      <input type="number" id="humidityLimit" value="50" min="0" max="400">
    </div>

    <button onclick="sendControl()">Αποστολή Εντολής</button>
  </div>

  <!-- Charts -->
  <div class="chart-container">
    <h2>Θερμοκρασία (°C)</h2>
    <canvas id="tempChart"></canvas>
  </div>

  <div class="chart-container">
    <h2>Υγρασία Αέρα (%)</h2>
    <canvas id="humChart"></canvas>
  </div>

  <div class="chart-container">
    <h2>Υγρασία Εδάφους</h2>
    <canvas id="soilChart"></canvas>
  </div>

  <div class="chart-container">
    <h2>Κατάσταση Αντλίας (On/Off)</h2>
    <canvas id="pumpChart"></canvas>
  </div>

  <script>
    const channelID = 3100035;
    const readAPIKey = "TBCHS5NBZ92WX8OJ";
    const writeAPIKey = "HZE91TDEY6NYQKDQ"; // για control fields
    const results = 30; // πόσα σημεία δείχνουμε

    // --- Fetch data για charts ---
    async function fetchField(field) {
      const url = `https://api.thingspeak.com/channels/${channelID}/fields/${field}.json?api_key=${readAPIKey}&results=${results}`;
      const response = await fetch(url);
      const data = await response.json();
      const labels = data.feeds.map(f => new Date(f.created_at).toLocaleTimeString());
      const values = data.feeds.map(f => parseFloat(f[`field${field}`]));
      return { labels, values };
    }

    async function drawCharts() {
      let tempData = await fetchField(1);
      new Chart(document.getElementById("tempChart"), {
        type: "line",
        data: {
          labels: tempData.labels,
          datasets: [{
            label: "Θερμοκρασία (°C)",
            data: tempData.values,
            borderColor: "red",
            fill: false
          }]
        }
      });

      let humData = await fetchField(2);
      new Chart(document.getElementById("humChart"), {
        type: "line",
        data: {
          labels: humData.labels,
          datasets: [{
            label: "Υγρασία Αέρα (%)",
            data: humData.values,
            borderColor: "blue",
            fill: false
          }]
        }
      });

      let soilData = await fetchField(3);
      new Chart(document.getElementById("soilChart"), {
        type: "line",
        data: {
          labels: soilData.labels,
          datasets: [{
            label: "Υγρασία Εδάφους",
            data: soilData.values,
            borderColor: "green",
            fill: false
          }]
        }
      });

      let pumpData = await fetchField(4);
      new Chart(document.getElementById("pumpChart"), {
        type: "bar",
        data: {
          labels: pumpData.labels,
          datasets: [{
            label: "Pump Status (0=Off, 1=On)",
            data: pumpData.values,
            backgroundColor: pumpData.values.map(v => v === 1 ? "orange" : "gray")
          }]
        },
        options: {
          scales: {
            y: {
              ticks: { stepSize: 1 },
              min: 0,
              max: 1
            }
          }
        }
      });
    }

    // --- Αποστολή εντολών στο ThingSpeak ---
    async function sendControl() {
      const pumpMode = document.getElementById("pumpMode").value;
      const pumpControl = document.querySelector("input[name='pumpControl']:checked").value;
      const humidityLimit = document.getElementById("humidityLimit").value;

      const url = `https://api.thingspeak.com/update?api_key=${writeAPIKey}&field5=${pumpMode}&field6=${pumpControl}&field7=${humidityLimit}`;
      await fetch(url);
      alert("Εντολή στάλθηκε στο Arduino!");
    }

    // --- Αυτόματο refresh κάθε 20s ---
    drawCharts();
    setInterval(drawCharts, 40000);
  
// --- Αποθήκευση επιλογών στο LocalStorage ---
function saveSettings() {
  const pumpMode = document.getElementById("pumpMode").value;
  const pumpControl = document.querySelector("input[name='pumpControl']:checked").value;
  const humidityLimit = document.getElementById("humidityLimit").value;

  localStorage.setItem("pumpMode", pumpMode);
  localStorage.setItem("pumpControl", pumpControl);
  localStorage.setItem("humidityLimit", humidityLimit);
}

// --- Επαναφορά επιλογών κατά το άνοιγμα της σελίδας ---
function loadSettings() {
  const pumpMode = localStorage.getItem("pumpMode");
  const pumpControl = localStorage.getItem("pumpControl");
  const humidityLimit = localStorage.getItem("humidityLimit");

  if (pumpMode !== null) {
    document.getElementById("pumpMode").value = pumpMode;
  }
  if (pumpControl !== null) {
    document.querySelector(`input[name='pumpControl'][value='${pumpControl}']`).checked = true;
  }
  if (humidityLimit !== null) {
    document.getElementById("humidityLimit").value = humidityLimit;
  }
}

// --- Συνδυασμός με αποστολή εντολής ---
const originalSendControl = sendControl;
sendControl = async function() {
  const pumpMode = document.getElementById("pumpMode").value;
  const pumpControl = document.querySelector("input[name='pumpControl']:checked").value;
  const humidityLimit = document.getElementById("humidityLimit").value;

  const url = `https://api.thingspeak.com/update?api_key=${writeAPIKey}&field5=${pumpMode}&field6=${pumpControl}&field7=${humidityLimit}`;
  await fetch(url);
  alert("Εντολή στάλθηκε στο Arduino!");

  // Αποθήκευση επιλογών
  saveSettings();
};

// --- Φόρτωση ρυθμίσεων μόλις ανοίξει η σελίδα ---
window.addEventListener("load", loadSettings);

</script>
</body>
</html>


