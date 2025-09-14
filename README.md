<!--
Simple single-file demo: Friend location sharing using HTML + Firebase Realtime Database + Leaflet
This modified version asks for:
 - "Your phone number" (the sharing person's phone) when starting to share
 - Optional "Track phone number" input to filter which friend's location to display

How to use:
 1. Create a Firebase project at https://console.firebase.google.com/
 2. In the project, enable Realtime Database and set rules for testing to:
      {
        "rules": {
          ".read": true,
          ".write": true
        }
      }
    (For production, secure these rules and use authentication.)
 3. Replace the firebaseConfig object below with your project's config values.
 4. Open this file in a browser (served via a local webserver is recommended — e.g. `npx http-server` or `python -m http.server`).

This demo:
 - asks the user for their phone number (used as identifier) to share their location
 - optionally allows entering a phone number to *track* (shows only that person's marker)
 - requests geolocation permission and writes {lat, lon, accuracy, timestamp, phone} to /locations/{phone}
 - listens for all locations and displays them on a Leaflet map with realtime updates; you can filter to a specific phone
 - lets the user start/stop sharing

WARNING: This example uses permissive DB rules for simplicity. Do not use these rules in production.
-->

<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Friend Location Share — Phone-based IDs (Demo)</title>
  <!-- Leaflet CSS -->
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" integrity="sha256-sA+4F9w6t1qkq8h3pF2v4rV9i3g3bq3FqWvYg9q0h2M=" crossorigin=""/>
  <style>
    body { font-family: system-ui, -apple-system, 'Segoe UI', Roboto, 'Helvetica Neue', Arial; margin: 0; }
    #topbar { padding: 12px; background:#0d6efd; color:white; display:flex; gap:8px; align-items:center; }
    #controls { margin-left:auto; display:flex; gap:8px; align-items:center; }
    #map { height: calc(100vh - 64px); }
    input, button { padding:8px; border-radius:6px; border:1px solid #ddd; }
    button { cursor:pointer; }
    .small { font-size:0.9rem; }
  </style>
</head>
<body>
  <div id="topbar">
    <div><strong>Friend Location Share — Demo</strong></div>
    <div id="status" class="small">Not connected</div>
    <div id="controls">
      <input id="myphone" placeholder="Your phone number (e.g. +919876543210)" />
      <input id="trackphone" placeholder="Track phone number (optional)" />
      <button id="start">Start Sharing</button>
      <button id="stop" disabled>Stop Sharing</button>
    </div>
  </div>

  <div id="map"></div>

  <!-- Firebase (compat SDK for quick demo) -->
  <script src="https://www.gstatic.com/firebasejs/9.22.2/firebase-app-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/9.22.2/firebase-database-compat.js"></script>

  <!-- Leaflet JS -->
  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js" integrity="sha256-o9N1j8k0t0XcBqJvWjz5o1z3eYf3KVp3b9g3kX3yjRk=" crossorigin=""></script>

  <script>
    // ====== REPLACE THIS CONFIG with YOUR Firebase project config ======
    const firebaseConfig = {
      apiKey: "REPLACE_WITH_YOUR_API_KEY",
      authDomain: "REPLACE_WITH_YOUR_AUTH_DOMAIN",
      databaseURL: "REPLACE_WITH_YOUR_DATABASE_URL",
      projectId: "REPLACE_WITH_YOUR_PROJECT_ID",
      storageBucket: "REPLACE_WITH_YOUR_STORAGE_BUCKET",
      messagingSenderId: "REPLACE_WITH_YOUR_SENDER_ID",
      appId: "REPLACE_WITH_YOUR_APP_ID"
    };
    // ==================================================================

    // Initialize Firebase
    firebase.initializeApp(firebaseConfig);
    const db = firebase.database();

    // UI elements
    const startBtn = document.getElementById('start');
    const stopBtn  = document.getElementById('stop');
    const statusEl = document.getElementById('status');
    const myPhoneEl = document.getElementById('myphone');
    const trackPhoneEl = document.getElementById('trackphone');

    let watchId = null;
    let myPhone = null;
    let markers = {}; // phone -> marker

    // Initialize map
    const map = L.map('map').setView([20.5937, 78.9629], 5); // center India by default
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution: '© OpenStreetMap contributors'
    }).addTo(map);

    // Helper: set status
    function setStatus(text) {
      statusEl.textContent = text;
    }

    // Start sharing location
    startBtn.addEventListener('click', async () => {
      myPhone = (myPhoneEl.value || '').trim();
      if (!myPhone) {
        alert('Please enter your phone number to identify your shared location.');
        return;
      }

      if (!navigator.geolocation) {
        alert('Geolocation is not supported by this browser.');
        return;
      }

      const locRef = db.ref('locations/' + encodeURIComponent(myPhone));

      watchId = navigator.geolocation.watchPosition(async (pos) => {
        const payload = {
          phone: myPhone,
          lat: pos.coords.latitude,
          lon: pos.coords.longitude,
          accuracy: pos.coords.accuracy || null,
          timestamp: pos.timestamp || Date.now()
        };
        await locRef.set(payload);
        setStatus('Sharing as: ' + myPhone + ' — last update: ' + new Date(payload.timestamp).toLocaleTimeString());
      }, (err) => {
        console.error('Geolocation error', err);
        alert('Geolocation error: ' + err.message);
      }, { enableHighAccuracy: true, maximumAge: 5000, timeout: 10000 });

      startBtn.disabled = true;
      stopBtn.disabled = false;
      myPhoneEl.disabled = true;
    });

    // Stop sharing location
    stopBtn.addEventListener('click', async () => {
      if (watchId !== null) {
        navigator.geolocation.clearWatch(watchId);
        watchId = null;
      }
      if (myPhone) {
        // Remove location from db (optional)
        await db.ref('locations/' + encodeURIComponent(myPhone)).remove();
      }
      setStatus('Not sharing');
      startBtn.disabled = false;
      stopBtn.disabled = true;
      myPhoneEl.disabled = false;
      myPhone = null;
    });

    // Listen for all locations and update markers in realtime
    const allLocRef = db.ref('locations');
    allLocRef.on('value', (snapshot) => {
      const data = snapshot.val() || {};

      // If user entered a phone to track, only show that marker (if present)
      const filterPhoneRaw = (trackPhoneEl.value || '').trim();
      const filterPhone = filterPhoneRaw ? encodeURIComponent(filterPhoneRaw) : null;

      // Remove markers for users no longer present or filtered out
      const currentIds = new Set(Object.keys(data));
      for (const phoneKey of Object.keys(markers)) {
        if (!currentIds.has(encodeURIComponent(phoneKey))) {
          map.removeLayer(markers[phoneKey]);
          delete markers[phoneKey];
        }
      }

      // Update markers
      Object.entries(data).forEach(([rawId, v]) => {
        const id = decodeURIComponent(rawId);
        if (!v || typeof v.lat !== 'number' || typeof v.lon !== 'number') return;

        // If filter specified and this isn't the filtered phone, skip
        if (filterPhone && rawId !== filterPhone) return;

        const latlng = [v.lat, v.lon];

        if (markers[id]) {
          markers[id].setLatLng(latlng);
          markers[id].getPopup().setContent(`<strong>${id}</strong><br>Updated: ${new Date(v.timestamp).toLocaleTimeString()}<br>Accuracy: ${v.accuracy || 'n/a'}`);
        } else {
          const mk = L.marker(latlng).addTo(map).bindPopup(`<strong>${id}</strong><br>Updated: ${new Date(v.timestamp).toLocaleTimeString()}<br>Accuracy: ${v.accuracy || 'n/a'}`);
          markers[id] = mk;
        }

        // If we've filtered to a single phone, center map on it
        if (filterPhone) {
          map.setView(latlng, 14);
        }
      });

      // If filtering and there's no data, show a message
      if (filterPhone && !data[filterPhone]) {
        setStatus('Tracking: ' + (trackPhoneEl.value || '') + ' — no live location available');
      } else if (!filterPhone) {
        setStatus('Showing all shared locations');
      }
    });

    // Re-run listener on filter input change to update view
    trackPhoneEl.addEventListener('input', () => {
      // The realtime listener handles filtering on each change; we can optionally adjust map view.
      // Small UX: if user clears filter, reset map view
      if (!trackPhoneEl.value.trim()) map.setView([20.5937, 78.9629], 5);
    });

    // Optional: click on map to center
    map.on('click', (e) => {
      map.setView(e.latlng, Math.max(12, map.getZoom()));
    });

    // Small UX: Enter from myPhone starts sharing
    myPhoneEl.addEventListener('keydown', (e) => {
      if (e.key === 'Enter') startBtn.click();
    });

    setStatus('Ready — enter your phone and click Start Sharing');
  </script>
</body>
</html>
