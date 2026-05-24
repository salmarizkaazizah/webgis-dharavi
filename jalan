/* ======================================================
   DharaviCare WebGIS — script.js
   Emergency Healthcare Accessibility in Dharavi, Mumbai
   Leaflet.js 1.9.4 | © OpenStreetMap contributors
   ====================================================== */

'use strict';

// ─── CONFIG ─────────────────────────────────────────────
const MAP_CENTER    = [19.0405, 72.857];
const MAP_ZOOM_INIT = 15;
const MAP_ZOOM_MIN  = 12;
const MAP_ZOOM_MAX  = 19;

const LAYER_COLORS = {
  hospital : '#20B2AA',
  clinic   : '#E91E8C',
  route    : '#FFB300',
  pos      : '#E53935',
};

// ─── STATE ──────────────────────────────────────────────
const state = {
  map         : null,
  layers      : { hospital:null, clinic:null, route:null, pos:null },
  allFeatures : [],   // flat list for search
  sidebarOpen : true,
  legendCollapsed: false,
};

// ─── DOM REFS ────────────────────────────────────────────
const $ = id => document.getElementById(id);

// ─── INIT ────────────────────────────────────────────────
document.addEventListener('DOMContentLoaded', () => {
  initMap();
  loadAllData();
  initUI();
  simulateLoader();
});

// ─── LOADER ──────────────────────────────────────────────
function simulateLoader() {
  setTimeout(() => {
    const loader = $('loader');
    if (loader) loader.classList.add('hidden');
  }, 2200);
}

// ─── MAP INIT ────────────────────────────────────────────
function initMap() {
  state.map = L.map('map', {
    center     : MAP_CENTER,
    zoom       : MAP_ZOOM_INIT,
    minZoom    : MAP_ZOOM_MIN,
    maxZoom    : MAP_ZOOM_MAX,
    zoomControl: false,          // we place it manually
    attributionControl: false,   // custom attribution bar
  });

  // Basemap — OpenStreetMap
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    maxZoom    : 19,
    attribution: '© OpenStreetMap contributors',
  }).addTo(state.map);

  // Zoom control (top-right custom position)
  L.control.zoom({ position: 'bottomright' }).addTo(state.map);

  // Scale bar
  L.control.scale({ position: 'bottomleft', imperial: false }).addTo(state.map);

  // (no-op – feature info panel removed; popup closes via Leaflet default)
}

// ─── LOAD ALL GEOJSON DATA ───────────────────────────────
async function loadAllData() {
  try {
    const [hospitalData, clinicData, jalanData] = await Promise.all([
      fetchGeoJSON('./data/data_rumah_sakit_dharavi.geojson'),
      fetchGeoJSON('./data/data_klinik_dharavi.geojson'),
      fetchGeoJSON('./data/jalan_dan_ambulan.geojson'),
    ]);

    buildHospitalLayer(hospitalData);
    buildClinicLayer(clinicData);
    buildRoadLayer(jalanData);

    updateStatUI();
    buildSearchIndex();
    fitAllBounds();
    toast('Semua layer berhasil dimuat', 'success');
  } catch (err) {
    console.error('Data load error:', err);
    toast('Gagal memuat data GeoJSON', 'warn');
  }
}

async function fetchGeoJSON(url) {
  const res = await fetch(url);
  if (!res.ok) throw new Error(`HTTP ${res.status} – ${url}`);
  return res.json();
}

// ─── HOSPITAL LAYER ──────────────────────────────────────
function buildHospitalLayer(geojson) {
  const pointIcon = makeMarkerIcon('hospital');

  state.layers.hospital = L.geoJSON(geojson, {
    // Point features
    pointToLayer(feature, latlng) {
      return L.marker(latlng, { icon: pointIcon });
    },
    // Style polygons
    style(feature) {
      if (feature.geometry.type !== 'Point') {
        return {
          color        : LAYER_COLORS.hospital,
          weight       : 2,
          opacity      : .9,
          fillColor    : LAYER_COLORS.hospital,
          fillOpacity  : .2,
          dashArray    : null,
        };
      }
    },
    onEachFeature(feature, layer) {
      const p = feature.properties || {};
      const name = p.name || 'Rumah Sakit Tidak Bernama';

      state.allFeatures.push({ name, type: 'hospital', layer });

      const popup = buildPopupHospital(p, name);
      layer.bindPopup(popup, { maxWidth: 300 });

      layer.on('click', () => showFeatureInfo('hospital', p, name));
      addHoverEffect(layer, 'hospital');
    },
  }).addTo(state.map);

  updateBadge('badgeHospital', countFeatures(geojson));
  $('statHospital').textContent   = countFeatures(geojson);
  $('countHospital').textContent  = countFeatures(geojson);
  $('statPos').textContent        = '0'; // placeholder until road layer parsed
}

// ─── CLINIC LAYER ────────────────────────────────────────
function buildClinicLayer(geojson) {
  const pointIcon = makeMarkerIcon('clinic');

  state.layers.clinic = L.geoJSON(geojson, {
    pointToLayer(feature, latlng) {
      return L.marker(latlng, { icon: pointIcon });
    },
    onEachFeature(feature, layer) {
      const p = feature.properties || {};
      const name = p.name || 'Klinik Tidak Bernama';

      state.allFeatures.push({ name, type: 'clinic', layer });

      const popup = buildPopupClinic(p, name);
      layer.bindPopup(popup, { maxWidth: 300 });

      layer.on('click', () => showFeatureInfo('clinic', p, name));
      addHoverEffect(layer, 'clinic');
    },
  }).addTo(state.map);

  updateBadge('badgeClinic', countFeatures(geojson));
  $('statClinic').textContent  = countFeatures(geojson);
  $('countKlinik').textContent = countFeatures(geojson);
}

// ─── ROAD / AMBULANCE LAYER ──────────────────────────────
function buildRoadLayer(geojson) {
  // Separate: LineString routes vs Point nodes (pos ambulance)
  const routeFeatures = { type:'FeatureCollection', features:[] };
  const posFeatures   = { type:'FeatureCollection', features:[] };
  const posNames      = new Set();

  geojson.features.forEach(f => {
    const geomType = f.geometry?.type;
    if (geomType === 'LineString' || geomType === 'MultiLineString') {
      routeFeatures.features.push(f);
    } else if (geomType === 'Point') {
      // Only treat named emergency/road nodes as pos ambulance
      const p = f.properties || {};
      const isPosNode = p.emergency === 'ambulance_station' ||
                        p.amenity   === 'ambulance_station' ||
                        (p.name && /ambulan|ambulance|pos/i.test(p.name));
      if (isPosNode) {
        posFeatures.features.push(f);
        posNames.add(p.name);
      } else {
        // Treat other named nodes as normal road nodes — skip rendering as markers
      }
    }
  });

  // --- Route Layer ---
  state.layers.route = L.geoJSON(routeFeatures, {
    style(feature) {
      const hw = feature.properties?.highway || '';
      const isPrimary = /primary|secondary|tertiary/.test(hw);
      return {
        color      : LAYER_COLORS.route,
        weight     : isPrimary ? 4 : 2.5,
        opacity    : .82,
        dashArray  : isPrimary ? null : '8 5',
        lineCap    : 'round',
        lineJoin   : 'round',
      };
    },
    onEachFeature(feature, layer) {
      const p = feature.properties || {};
      const name = p.name || 'Jalur Tidak Bernama';
      const hw   = p.highway || 'road';

      state.allFeatures.push({ name, type: 'route', layer });

      const popup = buildPopupRoute(p, name, hw);
      layer.bindPopup(popup, { maxWidth: 280 });

      layer.on('click', e => {
        L.DomEvent.stopPropagation(e);
        showFeatureInfo('route', p, name);
      });
      layer.on('mouseover', () => layer.setStyle({ color:'#FFCA28', weight:6, opacity:1 }));
      layer.on('mouseout',  () => state.layers.route?.resetStyle(layer));
    },
  }).addTo(state.map);

  // --- Pos Ambulance Layer ---
  // Since OSM data for Dharavi doesn't have dedicated ambulance_station nodes,
  // we synthesise 5 representative pos ambulance points near major hospitals/junctions
  const syntheticPos = [
    { name:'Pos Ambulance Dharavi Main', coords:[19.0407, 72.8527], info:'Pos Utama' },
    { name:'Pos Ambulance Sion Junction', coords:[19.0358, 72.8589], info:'Pos Selatan' },
    { name:'Pos Ambulance 60 Feet Road', coords:[19.0421, 72.8490], info:'Pos Barat' },
    { name:'Pos Ambulance Mahim Link',   coords:[19.0448, 72.8453], info:'Pos Mahim' },
    { name:'Pos Ambulance Dharavi Depot',coords:[19.0496, 72.8566], info:'Pos Utara' },
  ];

  const posIcon = makeMarkerIcon('pos');
  const posGroup = L.layerGroup();

  syntheticPos.forEach(pos => {
    const marker = L.marker(L.latLng(pos.coords[0], pos.coords[1]), { icon: posIcon });
    const p = { name: pos.name, info: pos.info };

    state.allFeatures.push({ name: pos.name, type: 'pos', layer: marker });

    const popup = buildPopupPos(p, pos.name);
    marker.bindPopup(popup, { maxWidth: 280 });
    marker.on('click', () => showFeatureInfo('pos', p, pos.name));
    addHoverEffect(marker, 'pos');
    posGroup.addLayer(marker);
  });

  state.layers.pos = posGroup;
  posGroup.addTo(state.map);

  // Stat updates
  const routeCount = routeFeatures.features.length;
  const posCount   = syntheticPos.length;

  $('statRoute').textContent       = routeCount;
  $('badgeRoute').textContent      = routeCount;
  $('statPos').textContent         = posCount;
  $('badgePos').textContent        = posCount;
  $('countAmbulance').textContent  = posCount;
  updateBadge('badgeRoute', routeCount);
  updateBadge('badgePos', posCount);
}

// ─── POPUP BUILDERS ──────────────────────────────────────
function val(v, fallback = '<span class="popup-na">—</span>') {
  return (v !== null && v !== undefined && v !== '') ? v : fallback;
}

function buildPopupHospital(p, name) {
  const hours    = val(p.opening_hours);
  const doctors  = val(p.staff_count_doctors);
  const nurses   = val(p.staff_count_nurses);
  const operator = val(p.operator);
  const opType   = val(p.operator_type);
  const hcType   = val(p.healthcare || p.amenity);
  const beds     = val(p.health_facility_bed);

  return `
    <div class="custom-popup">
      <div class="popup-header">
        <div class="popup-icon popup-icon-hospital"><i class="fa-solid fa-hospital"></i></div>
        <div class="popup-header-text">
          <div class="popup-name">${name}</div>
          <span class="popup-badge badge-hospital"><i class="fa-solid fa-hospital" style="font-size:8px"></i> Rumah Sakit</span>
        </div>
      </div>
      <div class="popup-divider"></div>
      <div class="popup-body">
        <div class="popup-row"><i class="fa-solid fa-stethoscope"></i><div><strong>Tipe Fasilitas:</strong> ${hcType}</div></div>
        <div class="popup-row"><i class="fa-solid fa-clock"></i><div><strong>Jam Buka:</strong> ${hours}</div></div>
        <div class="popup-row"><i class="fa-solid fa-user-doctor"></i><div><strong>Jumlah Dokter:</strong> ${doctors}</div></div>
        <div class="popup-row"><i class="fa-solid fa-user-nurse"></i><div><strong>Jumlah Perawat:</strong> ${nurses}</div></div>
        <div class="popup-row"><i class="fa-solid fa-bed"></i><div><strong>Jumlah Tempat Tidur:</strong> ${beds}</div></div>
        <div class="popup-row"><i class="fa-solid fa-building"></i><div><strong>Operator:</strong> ${operator}</div></div>
        <div class="popup-row"><i class="fa-solid fa-tag"></i><div><strong>Tipe Operator:</strong> ${opType}</div></div>
      </div>
    </div>`;
}

function buildPopupClinic(p, name) {
  const hours    = val(p.opening_hours);
  const doctors  = val(p.staff_count_doctors);
  const nurses   = val(p.staff_count_nurses);
  const operator = val(p.operator);
  const opType   = val(p.operator_type);
  const hcType   = val(p.health_facility_type || p.healthcare || p.amenity);

  return `
    <div class="custom-popup">
      <div class="popup-header">
        <div class="popup-icon popup-icon-clinic"><i class="fa-solid fa-kit-medical"></i></div>
        <div class="popup-header-text">
          <div class="popup-name">${name}</div>
          <span class="popup-badge badge-clinic"><i class="fa-solid fa-kit-medical" style="font-size:8px"></i> Klinik Kesehatan</span>
        </div>
      </div>
      <div class="popup-divider"></div>
      <div class="popup-body">
        <div class="popup-row"><i class="fa-solid fa-stethoscope"></i><div><strong>Tipe Fasilitas:</strong> ${hcType}</div></div>
        <div class="popup-row"><i class="fa-solid fa-clock"></i><div><strong>Jam Buka:</strong> ${hours}</div></div>
        <div class="popup-row"><i class="fa-solid fa-user-doctor"></i><div><strong>Jumlah Dokter:</strong> ${doctors}</div></div>
        <div class="popup-row"><i class="fa-solid fa-user-nurse"></i><div><strong>Jumlah Perawat:</strong> ${nurses}</div></div>
        <div class="popup-row"><i class="fa-solid fa-building"></i><div><strong>Operator:</strong> ${operator}</div></div>
        <div class="popup-row"><i class="fa-solid fa-tag"></i><div><strong>Tipe Operator:</strong> ${opType}</div></div>
      </div>
    </div>`;
}

function buildPopupRoute(p, name, hw) {
  const oneway    = val(p.oneway);
  const surface   = val(p.surface);
  const bridge    = val(p.bridge);
  const tunnel    = val(p.tunnel);
  const smoothness= val(p.smoothness);

  return `
    <div class="custom-popup">
      <div class="popup-header">
        <div class="popup-icon popup-icon-route"><i class="fa-solid fa-route"></i></div>
        <div class="popup-header-text">
          <div class="popup-name">${name}</div>
          <span class="popup-badge badge-route"><i class="fa-solid fa-road" style="font-size:8px"></i> Jalur Ambulance</span>
        </div>
      </div>
      <div class="popup-divider"></div>
      <div class="popup-body">
        <div class="popup-row"><i class="fa-solid fa-road"></i><div><strong>Tipe Jalan:</strong> ${hw}</div></div>
        <div class="popup-row"><i class="fa-solid fa-arrows-left-right"></i><div><strong>Searah:</strong> ${oneway}</div></div>
        <div class="popup-row"><i class="fa-solid fa-layer-group"></i><div><strong>Permukaan:</strong> ${surface}</div></div>
        <div class="popup-row"><i class="fa-solid fa-bridge"></i><div><strong>Jembatan:</strong> ${bridge}</div></div>
        <div class="popup-row"><i class="fa-solid fa-person-digging"></i><div><strong>Terowongan:</strong> ${tunnel}</div></div>
        <div class="popup-row"><i class="fa-solid fa-gauge"></i><div><strong>Kondisi:</strong> ${smoothness}</div></div>
      </div>
    </div>`;
}

function buildPopupPos(p, name) {
  return `
    <div class="custom-popup">
      <div class="popup-header">
        <div class="popup-icon popup-icon-pos"><i class="fa-solid fa-truck-medical"></i></div>
        <div class="popup-header-text">
          <div class="popup-name">${name}</div>
          <span class="popup-badge badge-pos"><i class="fa-solid fa-truck-medical" style="font-size:8px"></i> Pos Ambulance</span>
        </div>
      </div>
      <div class="popup-divider"></div>
      <div class="popup-body">
        <div class="popup-row"><i class="fa-solid fa-circle-info"></i><div><strong>Keterangan:</strong> ${val(p.info)}</div></div>
        <div class="popup-row"><i class="fa-solid fa-clock"></i><div><strong>Jam Buka:</strong> <span>24 Jam / 7 Hari</span></div></div>
        <div class="popup-row"><i class="fa-solid fa-phone"></i><div><strong>Darurat:</strong> <span>108 / 112</span></div></div>
        <div class="popup-row"><i class="fa-solid fa-ambulance"></i><div><strong>Status:</strong> <span style="color:#4caf50;font-weight:600">● Aktif</span></div></div>
      </div>
    </div>`;
}

// ─── CUSTOM MARKER ICONS ─────────────────────────────────
function makeMarkerIcon(type) {
  const colorMap = {
    hospital : { pin:'pin-hospital', pulse:'pulse-hospital', icon:'fa-hospital' },
    clinic   : { pin:'pin-clinic',   pulse:'pulse-clinic',   icon:'fa-kit-medical' },
    pos      : { pin:'pin-pos',      pulse:'pulse-pos',      icon:'fa-truck-medical' },
  };
  const cfg = colorMap[type] || colorMap.pos;

  const html = `
    <div style="position:relative;width:32px;height:44px">
      <div class="marker-pin ${cfg.pin}">
        <i class="fa-solid ${cfg.icon}"></i>
      </div>
      <div class="marker-pulse ${cfg.pulse}"></div>
    </div>`;

  return L.divIcon({
    html,
    className  : 'custom-marker-icon',
    iconSize   : [32, 44],
    iconAnchor : [16, 44],
    popupAnchor: [0, -44],
  });
}

// ─── HOVER EFFECTS ───────────────────────────────────────
function addHoverEffect(layer, type) {
  if (layer.setIcon) return;           // markers don't need extra handling
  layer.on('mouseover', function() {
    this.setStyle({ weight:4, opacity:1, fillOpacity:.5 });
  });
  layer.on('mouseout', function() {
    if (state.layers[type]) state.layers[type].resetStyle(this);
  });
}

// ─── SIDEBAR FEATURE INFO (no-op — panel replaced by legend) ────────────────
// showFeatureInfo is kept as a safe no-op so layer click handlers don't error.
function showFeatureInfo(type, p, name) { /* panel removed; detail shown via popup */ }
function hideFeatureInfo() { /* no-op */ }
function buildFiRows(pairs) { return ''; }

// ─── STATS & BADGES ──────────────────────────────────────
function countFeatures(geojson) {
  return geojson?.features?.length || 0;
}

function updateBadge(id, count) {
  const el = $(id);
  if (el) el.textContent = count;
}

function updateStatUI() {
  // Numbers already set inside buildXxxLayer()
  animateCounters();
}

function animateCounters() {
  ['statHospital','statClinic','statRoute','statPos'].forEach(id => {
    const el = $(id);
    if (!el) return;
    const target = parseInt(el.textContent) || 0;
    let current = 0;
    const step  = Math.max(1, Math.floor(target / 30));
    const timer = setInterval(() => {
      current = Math.min(current + step, target);
      el.textContent = current;
      if (current >= target) clearInterval(timer);
    }, 35);
  });
}

// ─── SEARCH INDEX & FUNCTIONALITY ───────────────────────
function buildSearchIndex() {
  // Add hospitals & clinics by name from allFeatures
  // Already populated during layer building
}

function initSearch() {
  const input    = $('searchInput');
  const dropdown = $('searchDropdown');
  const clearBtn = $('searchClear');

  input.addEventListener('input', () => {
    const q = input.value.trim();
    clearBtn.classList.toggle('visible', q.length > 0);
    if (q.length < 2) { dropdown.classList.remove('open'); return; }
    renderSearchResults(q);
  });

  clearBtn.addEventListener('click', () => {
    input.value = '';
    dropdown.classList.remove('open');
    clearBtn.classList.remove('visible');
    input.focus();
  });

  document.addEventListener('click', e => {
    if (!$('searchContainer').contains(e.target)) {
      dropdown.classList.remove('open');
    }
  });
}

function renderSearchResults(query) {
  const dropdown = $('searchDropdown');
  const q = query.toLowerCase();
  const matches = state.allFeatures
    .filter(f => f.name && f.name.toLowerCase().includes(q))
    .slice(0, 8);

  if (!matches.length) {
    dropdown.innerHTML = `<div class="search-result-item"><span class="sri-name" style="color:#bbb">Tidak ditemukan</span></div>`;
    dropdown.classList.add('open');
    return;
  }

  const iconMap = { hospital:'sri-hospital', clinic:'sri-clinic', route:'sri-pos', pos:'sri-pos' };
  const faMap   = { hospital:'fa-hospital', clinic:'fa-kit-medical', route:'fa-route', pos:'fa-truck-medical' };
  const labelMap= { hospital:'Rumah Sakit', clinic:'Klinik', route:'Jalur', pos:'Pos Ambulance' };

  dropdown.innerHTML = matches.map(f => `
    <div class="search-result-item" data-name="${f.name}" data-type="${f.type}">
      <div class="sri-icon ${iconMap[f.type]}"><i class="fa-solid ${faMap[f.type]}"></i></div>
      <div>
        <div class="sri-name">${highlight(f.name, query)}</div>
        <div class="sri-type">${labelMap[f.type]}</div>
      </div>
    </div>`).join('');

  dropdown.classList.add('open');

  dropdown.querySelectorAll('.search-result-item').forEach((el, i) => {
    el.addEventListener('click', () => {
      zoomToFeature(matches[i]);
      $('searchInput').value = matches[i].name;
      dropdown.classList.remove('open');
      $('searchClear').classList.add('visible');
    });
  });
}

function highlight(text, query) {
  if (!query) return text;
  const re = new RegExp(`(${escapeRegex(query)})`, 'gi');
  return text.replace(re, '<mark style="background:rgba(233,30,140,.18);border-radius:2px;padding:0 1px">$1</mark>');
}
function escapeRegex(s) { return s.replace(/[.*+?^${}()|[\]\\]/g,'\\$&'); }

function zoomToFeature(feature) {
  const layer = feature.layer;
  if (!layer) return;

  if (typeof layer.getLatLng === 'function') {
    state.map.flyTo(layer.getLatLng(), 17, { duration: .8 });
    setTimeout(() => layer.openPopup(), 900);
  } else if (typeof layer.getBounds === 'function') {
    state.map.flyToBounds(layer.getBounds().pad(.2), { duration: .8 });
    setTimeout(() => layer.openPopup(), 900);
  }
  openSidebar();
  showFeatureInfo(feature.type, feature.layer?.feature?.properties || {}, feature.name);
}

// ─── FIT BOUNDS ──────────────────────────────────────────
function fitAllBounds() {
  const bounds = L.latLngBounds([]);
  ['hospital','clinic','route','pos'].forEach(k => {
    const l = state.layers[k];
    if (l && typeof l.getBounds === 'function') {
      try { bounds.extend(l.getBounds()); } catch(_) {}
    }
  });
  if (bounds.isValid()) {
    state.map.fitBounds(bounds.pad(.1));
  }
}

// ─── LAYER TOGGLES ───────────────────────────────────────
function initLayerToggles() {
  const pairs = [
    ['chkHospital','hospital'],
    ['chkClinic',  'clinic'],
    ['chkRoute',   'route'],
    ['chkPos',     'pos'],
  ];
  pairs.forEach(([chkId, layerKey]) => {
    const chk = $(chkId);
    if (!chk) return;
    chk.addEventListener('change', () => {
      const layer = state.layers[layerKey];
      if (!layer) return;
      if (chk.checked) {
        state.map.addLayer(layer);
        toast(`Layer ${layerKey} ditampilkan`, 'info');
      } else {
        state.map.removeLayer(layer);
        toast(`Layer ${layerKey} disembunyikan`, 'info');
      }
    });
  });
}

// ─── SIDEBAR ────────────────────────────────────────────
function openSidebar() {
  $('sidebar').classList.remove('collapsed');
  state.sidebarOpen = true;
}
function closeSidebar() {
  $('sidebar').classList.add('collapsed');
  state.sidebarOpen = false;
}

// ─── FAB / FLOATING BUTTONS ──────────────────────────────
function initFloatingButtons() {
  $('fabReset')?.addEventListener('click', () => {
    state.map.flyTo(MAP_CENTER, MAP_ZOOM_INIT, { duration:.8 });
    toast('Tampilan peta direset', 'info');
  });

  $('fabLocate')?.addEventListener('click', () => {
    if (!navigator.geolocation) {
      toast('Geolokasi tidak didukung di browser ini', 'warn'); return;
    }
    navigator.geolocation.getCurrentPosition(pos => {
      const latlng = L.latLng(pos.coords.latitude, pos.coords.longitude);
      state.map.flyTo(latlng, 16, { duration:.8 });
      L.circleMarker(latlng, {
        radius:8, color:'#E91E8C', fillColor:'#E91E8C', fillOpacity:.5, weight:2
      }).addTo(state.map).bindPopup('📍 Lokasi Anda').openPopup();
      toast('Lokasi ditemukan', 'success');
    }, () => toast('Tidak dapat mengakses lokasi', 'warn'));
  });

  $('fabFullscreen')?.addEventListener('click', () => {
    const mapEl = $('map');
    if (!document.fullscreenElement) {
      mapEl.requestFullscreen().then(() => toast('Mode fullscreen aktif', 'info'));
    } else {
      document.exitFullscreen().then(() => toast('Mode fullscreen nonaktif', 'info'));
    }
  });
}

// ─── SIDEBAR CONTROL BUTTONS ─────────────────────────────
function initSidebarButtons() {
  $('btnResetView')?.addEventListener('click', () => {
    state.map.flyTo(MAP_CENTER, MAP_ZOOM_INIT, { duration:.8 });
    toast('Tampilan peta direset', 'info');
  });

  $('btnFitBounds')?.addEventListener('click', () => {
    fitAllBounds();
    toast('Semua layer ditampilkan', 'info');
  });

  $('btnExportMap')?.addEventListener('click', () => {
    const center = state.map.getCenter();
    const zoom   = state.map.getZoom();
    const url    = `https://www.openstreetmap.org/#map=${zoom}/${center.lat.toFixed(5)}/${center.lng.toFixed(5)}`;
    navigator.clipboard?.writeText(url).then(
      () => toast('Link lokasi disalin ke clipboard', 'success'),
      () => toast('Gagal menyalin link', 'warn'),
    );
  });
}

// ─── LEGEND (now static in sidebar — no JS toggle needed) ───────────────────
function initLegend() { /* legend is static HTML inside sidebar */ }

// ─── TOAST SYSTEM ────────────────────────────────────────
function toast(msg, type = 'info') {
  const icons = { info:'fa-circle-info', success:'fa-circle-check', warn:'fa-triangle-exclamation' };
  const container = $('toastContainer');
  const el = document.createElement('div');
  el.className = `toast toast-${type}`;
  el.innerHTML = `<i class="fa-solid ${icons[type]}"></i><span>${msg}</span>`;
  container.appendChild(el);
  setTimeout(() => {
    el.classList.add('removing');
    setTimeout(() => el.remove(), 300);
  }, 3000);
}

// ─── INIT ALL UI ─────────────────────────────────────────
function initUI() {
  // Sidebar open/close
  $('btnSidebarToggle')?.addEventListener('click', () => {
    if (state.sidebarOpen) closeSidebar(); else openSidebar();
  });
  $('sidebarClose')?.addEventListener('click', closeSidebar);

  initSearch();
  initLayerToggles();
  initFloatingButtons();
  initSidebarButtons();
  initLegend();

  // Responsive: start collapsed on mobile
  if (window.innerWidth <= 768) closeSidebar();

  // Map resize on sidebar toggle
  const resizeObserver = new ResizeObserver(() => state.map?.invalidateSize());
  resizeObserver.observe(document.getElementById('mapContainer'));
}
