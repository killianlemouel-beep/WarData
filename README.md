<!DOCTYPE html>
<html>
<head>
  <title>WarData</title>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
  <style>
    body { margin: 0; padding: 0; font-family: Arial, sans-serif; }
    #map { width: 100vw; height: 100vh; }
    #toggle-settings {
      position: absolute; top: 10px; left: 10px; z-index: 1001;
      width: 40px; height: 40px; background: white; border-radius: 5px;
      border: none; box-shadow: 0 0 10px rgba(0,0,0,0.2); font-size: 20px;
      transition: transform 0.3s ease;
    }
    #toggle-settings.rotate { transform: rotate(90deg); }
    #settings-panel {
      position: absolute; top: 10px; left: 50px; z-index: 1000;
      background: white; padding: 10px; border-radius: 5px;
      box-shadow: 0 0 10px rgba(0,0,0,0.2); max-width: 200px;
      transition: transform 0.3s ease; transform: translateX(0);
    }
    #settings-panel.hidden { transform: translateX(-100%); }
    .legend {
      position: absolute; bottom: 20px; right: 20px; z-index: 1000;
      background: white; padding: 10px; border-radius: 5px;
      box-shadow: 0 0 10px rgba(0,0,0,0.2);
    }
    .battle-popup { min-width: 250px; max-width: 300px; }
    .battle-title { font-weight: bold; font-size: 16px; }
    .battle-title.camp { color: #000; }
    .battle-title.battle { color: #d32f2f; }
    .casualties-table { width: 100%; margin-top: 10px; border-collapse: collapse; }
    .casualties-table th, .casualties-table td {
      border: 1px solid #ddd; padding: 4px; text-align: center;
    }
    .casualties-table th { background: #f2f2f2; }
    .total-row { font-weight: bold; background: #f8f8f8; }
    hr { margin: 8px 0; }
  </style>
</head>
<body>
  <div id="map"></div>

  <!-- Bouton pour ouvrir/fermer le menu -->
  <button id="toggle-settings">⚙️</button>

  <!-- Menu paramètre rabattable -->
  <div class="control-panel" id="settings-panel">
    <select id="language-selector" onchange="changeLanguage(this.value)">
      <option value="fr" selected>Français</option>
      <option value="en">English</option>
    </select>
    <select id="era-selector" onchange="changeEra(this.value)">
      <option value="ww2" selected>Seconde Guerre mondiale</option>
    </select>
    <button onclick="resetView()" id="reset-btn">Réinitialiser</button>
  </div>

  <!-- Légende -->
  <div class="legend">
    <div><strong id="legend-title">Légende</strong></div>
    <div><span style="color: red;">●</span> <span id="legend-battle">Bataille</span></div>
    <div><span style="color: orange;">●</span> <span id="legend-subbattle">Sous-bataille</span></div>
    <div><span style="color: black;">●</span> <span id="legend-camp">Camp d'extermination</span></div>
  </div>

  <script>
    // ========== CONFIGURATION INITIALE ==========
    let currentLanguage = "fr";
    let currentEra = "ww2";
    let map;
    let battleMarkers = {};
    let subBattleMarkers = {};
    let activeBattle = null;
    let settingsOpen = true;

    // ========== TRADUCTIONS ==========
    const translations = {
      fr: {
        title: "WarData",
        battle: "Bataille",
        subBattle: "Sous-bataille",
        camp: "Camp d'extermination",
        deaths: "Morts",
        wounded: "Blessés",
        total: "Total",
        year: "Année",
        date: "Date",
        description: "Description",
        casualties: "Pertes par nationalité",
        nation: "Nation",
        reset: "Réinitialiser",
        legend: "Légende",
        era_ww2: "Seconde Guerre mondiale"
      },
      en: {
        title: "WarData",
        battle: "Battle",
        subBattle: "Sub-battle",
        camp: "Extermination Camp",
        deaths: "Deaths",
        wounded: "Wounded",
        total: "Total",
        year: "Year",
        date: "Date",
        description: "Description",
        casualties: "Casualties by nationality",
        nation: "Nation",
        reset: "Reset",
        legend: "Legend",
        era_ww2: "World War II"
      }
    };

    // ========== DONNÉES ==========
    const eras = {
      ww2: {
        name: { fr: "Seconde Guerre mondiale", en: "World War II" },
        startYear: 1939,
        endYear: 1945,
        battles: [
          {
            id: "stalingrad",
            name: { fr: "Bataille de Stalingrad", en: "Battle of Stalingrad" },
            lat: 48.7194,
            lng: 44.5018,
            deaths: 1995000,
            wounded: 1120000,
            year: 1942,
            date: { fr: "23 août 1942 - 2 février 1943", en: "August 23, 1942 - February 2, 1943" },
            description: {
              fr: "Siège majeur entre l'URSS et l'Allemagne nazie. Tournant de la guerre à l'Est.",
              en: "Major siege between the USSR and Nazi Germany. Turning point of the war in the East."
            },
            casualtiesByCountry: {
              URSS: { deaths: 1129000, wounded: 651000 },
              Allemagne: { deaths: 850000, wounded: 470000 },
              Roumanie: { deaths: 50000, wounded: 30000 },
              Italie: { deaths: 15000, wounded: 10000 }
            },
            subBattles: [
              {
                name: { fr: "Opération Uranus", en: "Operation Uranus" },
                lat: 48.8,
                lng: 44.6,
                deaths: 850000,
                wounded: 500000,
                date: { fr: "19-23 novembre 1942", en: "Nov 19-23, 1942" },
                description: {
                  fr: "Contre-offensive soviétique qui encercle la 6e armée allemande.",
                  en: "Soviet counter-offensive that encircled the German 6th Army."
                }
              },
              {
                name: { fr: "Opération Saturne", en: "Operation Saturn" },
                lat: 48.5,
                lng: 44.2,
                deaths: 200000,
                wounded: 120000,
                date: { fr: "Décembre 1942", en: "December 1942" },
                description: {
                  fr: "Tentative d'élargir l'encerclement vers Rostov.",
                  en: "Attempt to expand the encirclement towards Rostov."
                }
              },
              {
                name: { fr: "Opération Koltso", en: "Operation Ring" },
                lat: 48.7,
                lng: 44.5,
                deaths: 250000,
                wounded: 150000,
                date: { fr: "Janvier-Février 1943", en: "Jan-Feb 1943" },
                description: {
                  fr: "Destruction des forces allemandes encerclées.",
                  en: "Destruction of the encircled German forces."
                }
              }
            ]
          },
          {
            id: "normandy",
            name: { fr: "Débarquement de Normandie", en: "Normandy Landings" },
            lat: 49.4142,
            lng: -0.6016,
            deaths: 425000,
            wounded: 250000,
            year: 1944,
            date: { fr: "6 juin - 30 août 1944", en: "June 6 - August 30, 1944" },
            description: {
              fr: "Opération Overlord : début de la libération de l'Europe de l'Ouest.",
              en: "Operation Overlord: beginning of the liberation of Western Europe."
            },
            casualtiesByCountry: {
              USA: { deaths: 29000, wounded: 106000 },
              "Royaume-Uni": { deaths: 11000, wounded: 54000 },
              Canada: { deaths: 5000, wounded: 13000 },
              Allemagne: { deaths: 200000, wounded: 200000 },
              France: { deaths: 20000, wounded: 19000 }
            },
            subBattles: [
              {
                name: { fr: "Plage Utah", en: "Utah Beach" },
                lat: 49.44,
                lng: -1.10,
                deaths: 197,
                wounded: 600,
                date: { fr: "6 juin 1944", en: "June 6, 1944" },
                description: {
                  fr: "Débarquement américain. Moins de résistance allemande.",
                  en: "American landing. Less German resistance."
                }
              },
              {
                name: { fr: "Plage Omaha", en: "Omaha Beach" },
                lat: 49.37,
                lng: -0.93,
                deaths: 2000,
                wounded: 5000,
                date: { fr: "6 juin 1944", en: "June 6, 1944" },
                description: {
                  fr: "Plage la plus meurtrière pour les Alliés.",
                  en: "Deadliest beach for the Allies."
                }
              }
            ]
          },
          {
            id: "berlin",
            name: { fr: "Bataille de Berlin", en: "Battle of Berlin" },
            lat: 52.5200,
            lng: 13.4050,
            deaths: 811000,
            wounded: 1200000,
            year: 1945,
            date: { fr: "16 avril - 2 mai 1945", en: "April 16 - May 2, 1945" },
            description: {
              fr: "Chute du Troisième Reich. Dernière grande bataille en Europe.",
              en: "Fall of the Third Reich. Last major battle in Europe."
            },
            casualtiesByCountry: {
              URSS: { deaths: 361000, wounded: 1100000 },
              Allemagne: { deaths: 450000, wounded: 100000 },
              Pologne: { deaths: 10000, wounded: 20000 }
            }
          },
          // ===== CAMPS INTÉGRÉS =====
          {
            id: "auschwitz",
            name: { fr: "Auschwitz-Birkenau", en: "Auschwitz-Birkenau" },
            lat: 50.0275,
            lng: 19.1787,
            deaths: 1100000,
            wounded: 0,
            year: 1940,
            date: { fr: "Mai 1940 - Janvier 1945", en: "May 1940 - January 1945" },
            description: {
              fr: "Plus grand camp d'extermination nazi. 1,1 million de victimes, dont 90% de Juifs.",
              en: "Largest Nazi extermination camp. 1.1 million victims, 90% of whom were Jews."
            },
            casualtiesByCountry: {
              Juif: { deaths: 990000, wounded: 0 },
              Polonais: { deaths: 70000, wounded: 0 },
              Tziganes: { deaths: 20000, wounded: 0 },
              "Prisonniers de guerre": { deaths: 15000, wounded: 0 }
            },
            isCamp: true
          },
          {
            id: "treblinka",
            name: { fr: "Treblinka", en: "Treblinka" },
            lat: 52.6442,
            lng: 22.0733,
            deaths: 800000,
            wounded: 0,
            year: 1942,
            date: { fr: "Juillet 1942 - Octobre 1943", en: "July 1942 - October 1943" },
            description: {
              fr: "Camp d'extermination en Pologne. Environ 800 000 Juifs assassinés.",
              en: "Extermination camp in Poland. Approximately 800,000 Jews murdered."
            },
            casualtiesByCountry: {
              Juif: { deaths: 800000, wounded: 0 }
            },
            isCamp: true
          }
        ]
      }
    };

    // ========== FONCTIONS DE TRADUCTION ==========
    function t(key) {
      return translations[currentLanguage][key] || key;
    }

    function updateUI() {
      document.title = t("title");
      document.getElementById("legend-title").textContent = t("legend");
      document.getElementById("legend-battle").textContent = t("battle");
      document.getElementById("legend-subbattle").textContent = t("subBattle");
      document.getElementById("legend-camp").textContent = t("camp");
      document.getElementById("reset-btn").textContent = t("reset");
      document.getElementById("era-selector").innerHTML = `
        <select onchange="changeEra(this.value)">
          <option value="ww2" selected>${eras.ww2.name[currentLanguage]}</option>
        </select>
      `;
    }

    function changeLanguage(lang) {
      currentLanguage = lang;
      updateUI();
      updateAllPopups();
    }

    // ========== FONCTIONS DE CARTE ==========
    function initMap() {
      map = L.map('map').setView([20, 0], 2);
      L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a>'
      }).addTo(map);
    }

    function resetView() {
      if (activeBattle) {
        hideSubBattles(activeBattle);
        activeBattle = null;
      }
      const eraData = eras[currentEra];
      const allLocations = eraData.battles.map(b => [b.lat, b.lng]);
      map.fitBounds(allLocations);
    }

    function changeEra(era) {
      currentEra = era;
      clearMap();
      loadEraData();
      resetView();
    }

    function clearMap() {
      Object.values(battleMarkers).forEach(marker => map.removeLayer(marker));
      Object.values(subBattleMarkers).forEach(markers => markers.forEach(m => map.removeLayer(m)));
      battleMarkers = {};
      subBattleMarkers = {};
    }

    function loadEraData() {
      const eraData = eras[currentEra];
      const maxDeaths = Math.max(...eraData.battles.map(b => b.deaths));

      eraData.battles.forEach(location => {
        const radius = 5 + (location.deaths / maxDeaths) * 50;
        const fillColor = location.isCamp ? '#000000' : '#ff0000';

        const marker = L.circleMarker([location.lat, location.lng], {
          radius: radius,
          fillColor: fillColor,
          color: '#000',
          weight: 1,
          fillOpacity: 0.7
        }).addTo(map);

        if (!location.isCamp) {
          marker.on('click', () => {
            if (activeBattle === location.id) {
              hideSubBattles(location.id);
              activeBattle = null;
            } else {
              if (activeBattle) hideSubBattles(activeBattle);
              if (location.subBattles) {
                showSubBattles(location, maxDeaths);
                activeBattle = location.id;
              }
            }
          });
        }

        marker.bindPopup(createLocationPopup(location));
        battleMarkers[location.id] = marker;
      });
    }

    function showSubBattles(battle, maxDeaths) {
      if (!battle.subBattles || battle.subBattles.length === 0) return;

      subBattleMarkers[battle.id] = battle.subBattles.map(sub => {
        const radius = 3 + (sub.deaths / maxDeaths) * 30;
        return L.circleMarker([sub.lat, sub.lng], {
          radius: radius,
          fillColor: '#ff9900',
          color: '#000',
          weight: 1,
          fillOpacity: 0.8
        })
        .addTo(map)
        .bindPopup(createSubBattlePopup(sub));
      });
    }

    function hideSubBattles(battleId) {
      if (subBattleMarkers[battleId]) {
        subBattleMarkers[battleId].forEach(marker => map.removeLayer(marker));
        delete subBattleMarkers[battleId];
      }
    }

    function updateAllPopups() {
      Object.values(battleMarkers).forEach(marker => {
        const battle = eras[currentEra].battles.find(b => b.id === marker.options.id);
        if (battle) marker.setPopupContent(createLocationPopup(battle));
      });
    }

    // ========== CRÉATION DES POPUPS ==========
    function createLocationPopup(location) {
      const totalDeaths = location.deaths + (location.subBattles ?
        location.subBattles.reduce((sum, sub) => sum + sub.deaths, 0) : 0);
      const totalWounded = location.wounded || 0;

      const titleClass = location.isCamp ? 'camp' : 'battle';

      let html = `
        <div class="battle-popup">
          <div class="battle-title ${titleClass}">${location.name[currentLanguage]}</div>
          <div>📅 ${location.date[currentLanguage]} | 💀 ${totalDeaths.toLocaleString()} ${t("deaths")}`
          ;

      if (!location.isCamp && totalWounded > 0) {
        html += ` | 🩹 ${totalWounded.toLocaleString()} ${t("wounded")}`;
      }

      html += `</div>
          <p style="margin: 10px 0; font-size: 14px;">${location.description[currentLanguage]}</p>
      `;

      if (location.casualtiesByCountry) {
        html += `<hr><div><strong>${t("casualties")}</strong></div>`;
        html += `<table class="casualties-table">
          <tr>
            <th>${t("nation")}</th>
            <th>${t("deaths")}</th>`;

        if (!location.isCamp) {
          html += `<th>${t("wounded")}</th>`;
        }

        html += `</tr>`;

        for (const [group, stats] of Object.entries(location.casualtiesByCountry)) {
          html += `<tr><td>${group}</td><td>${stats.deaths.toLocaleString()}</td>`;
          if (!location.isCamp) {
            html += `<td>${stats.wounded.toLocaleString()}</td>`;
          }
          html += `</tr>`;
        }

        html += `
          <tr class="total-row">
            <td><strong>${t("total")}</strong></td>
            <td><strong>${totalDeaths.toLocaleString()}</strong></td>`;

        if (!location.isCamp) {
          html += `<td><strong>${totalWounded.toLocaleString()}</strong></td>`;
        }

        html += `</tr></table>`;
      }

      if (!location.isCamp && location.subBattles && location.subBattles.length > 0) {
        html += `<hr><div><strong>${t("subBattle")}s:</strong> <span style="color: #666;">(Cliquez sur la bataille pour les afficher sur la carte)</span></div>`;
        html += `<ul style="list-style-type: none; padding: 0; margin: 5px 0;">`;
        location.subBattles.forEach(sub => {
          html += `
            <li style="margin: 5px 0; padding: 5px; background: #f8f8f8; border-radius: 3px;">
              🎯 ${sub.name[currentLanguage]} (${sub.deaths.toLocaleString()} ${t("deaths")})
            </li>
          `;
        });
        html += `</ul>`;
      }

      html += `</div>`;
      return html;
    }

    function createSubBattlePopup(sub) {
      return `
        <div class="battle-popup">
          <div class="battle-title battle">${sub.name[currentLanguage]}</div>
          <div>📅 ${sub.date[currentLanguage]} | 💀 ${sub.deaths.toLocaleString()} ${t("deaths")} | 🩹 ${sub.wounded.toLocaleString()} ${t("wounded")}</div>
          <p style="margin: 10px 0; font-size: 14px;">${sub.description[currentLanguage]}</p>
        </div>
      `;
    }

    // ========== GESTION DU MENU RABATTABLE ==========
    function toggleSettings() {
      const panel = document.getElementById('settings-panel');
      const button = document.getElementById('toggle-settings');

      if (settingsOpen) {
        panel.classList.add('hidden');
        button.classList.add('rotate');
      } else {
        panel.classList.remove('hidden');
        button.classList.remove('rotate');
      }
      settingsOpen = !settingsOpen;
    }

    // ========== INITIALISATION ==========
    window.onload = function() {
      initMap();
      updateUI();
      loadEraData();
      document.getElementById('toggle-settings').addEventListener('click', toggleSettings);
    };
  </script>
</body>
</html>
