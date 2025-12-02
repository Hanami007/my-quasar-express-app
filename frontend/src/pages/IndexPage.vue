<template>
  <div class="max-w-6xl mx-auto p-4">
    <header class="mb-4">
      <h1 class="text-2xl font-semibold">Longan Map — แผนที่ (Northern Thailand)</h1>
      <p class="text-sm text-slate-600">เพิ่มตำแหน่งด้วยฟอร์ม แคชพิกัดใน localStorage ใช้ Leaflet + OpenStreetMap + Nominatim</p>
    </header>

    <section class="grid grid-cols-1 lg:grid-cols-3 gap-4">
      <!-- Form column -->
      <div class="col-span-1 bg-white rounded-lg shadow p-4">
        <h2 class="font-medium mb-2">เพิ่ม / แก้ไขจุด (Add or edit marker)</h2>

        <form id="markerForm" class="space-y-2">
          <input type="hidden" id="editingId" />

          <div>
            <label class="block text-sm">สถานที่ (Place)</label>
            <input id="place" class="w-full border rounded px-2 py-1" placeholder="ตัวอย่าง: ตลาดสันทราย" />
          </div>

          <div>
            <label class="block text-sm">ชื่อ-สกุล (ชื่อผู้ติดต่อ)</label>
            <input id="fullname" class="w-full border rounded px-2 py-1" placeholder="ตัวอย่าง: นายธราเทพ" />
          </div>

          <div>
            <label class="block text-sm">จังหวัด (Province)</label>
            <select id="province" class="w-full border rounded px-2 py-1">
              <option value="">-- เลือกจังหวัด (Northern) --</option>
              <option>เชียงใหม่</option>
              <option>ลำพูน</option>
              <option>ลำปาง</option>
              <option>เชียงราย</option>
              <option>แพร่</option>
              <option>น่าน</option>
              <option>พะเยา</option>
              <option>แม่ฮ่องสอน</option>
            </select>
          </div>

          <div class="grid grid-cols-2 gap-2">
            <div>
              <label class="block text-sm">ละติจูด (Latitude)</label>
              <input id="lat" class="w-full border rounded px-2 py-1" placeholder="เช่น 18.7883" />
            </div>
            <div>
              <label class="block text-sm">ลองจิจูด (Longitude)</label>
              <input id="lng" class="w-full border rounded px-2 py-1" placeholder="เช่น 98.9853" />
            </div>
          </div>

          

          <div class="flex gap-2 mt-2">
            <button type="submit" id="addBtn" class="bg-emerald-600 text-white px-3 py-1 rounded">เพิ่ม / บันทึก</button>
            <button type="button" id="clearBtn" class="bg-slate-200 px-3 py-1 rounded">ล้าง</button>
          </div>

          <p class="text-xs text-slate-500 mt-2">หมายเหตุ: หากไม่กรอกพิกัด ระบบจะพยายามแปลงชื่อสถานที่เป็นพิกัด (Geocoding) ผ่าน Nominatim และจะเก็บผลลัพธ์ไว้ที่ localStorage เพื่อลดการเรียกซ้ำ</p>
        </form>

        <hr class="my-3" />

        <div>
          <h3 class="font-medium">ตัวอย่าง</h3>
          <ol class="list-decimal pl-5 text-sm text-slate-700 mt-2">
            <li>เลือกจังหวัด "เชียงใหม่" จะมีตัวอย่างพิกัดและข้อมูล</li>
            <li>หรือกรอกพิกัดเองแล้วกด "เพิ่ม / บันทึก"</li>
          </ol>
        </div>

      </div>

      <!-- Map column -->
      <div class="col-span-2 lg:col-span-2 bg-white rounded-lg shadow p-2">
        <div id="map" style="height:70vh; min-height:420px;"></div>
      </div>
    </section>

    <footer class="mt-4 text-sm text-slate-600">ไฟล์: <strong>IndexPage.vue</strong> — พัฒนาโดย Vue + JavaScript + Leaflet</footer>
  </div>
</template>

<script setup>
import { onMounted } from 'vue';

// We'll dynamically load Tailwind (cdn) + Leaflet assets so this page can work
const loadScript = (src) => new Promise((res, rej) => {
  if (document.querySelector(`script[src="${src}"]`)) return res();
  const s = document.createElement('script'); s.src = src; s.onload = res; s.onerror = rej; document.head.appendChild(s);
});
const loadCss = (href) => {
  if (document.querySelector(`link[href="${href}"]`)) return;
  const l = document.createElement('link'); l.rel = 'stylesheet'; l.href = href; document.head.appendChild(l);
};

onMounted(async () => {
  // inject Tailwind CDN (optional)
  try { await loadScript('https://cdn.tailwindcss.com'); } catch (e) { /* ignore */ }

  // inject Leaflet CSS and JS
  loadCss('https://unpkg.com/leaflet@1.9.4/dist/leaflet.css');
  try { await loadScript('https://unpkg.com/leaflet@1.9.4/dist/leaflet.js'); } catch (e) { console.warn('Leaflet load failed', e); }

  // now run the original page logic, using global `L`
  try {
    const L = window.L;
    if (!L) { console.warn('Leaflet not available'); return; }

    // ----------------------- Configuration -----------------------
    const STORAGE_KEY = 'longan_markers_v1';
    const GEOCODE_CACHE_KEY = 'longan_geocode_cache_v1';
    const NOMINATIM_EMAIL = 'example@example.com';
    const MAP_CENTER = [18.5, 98.5];
    const MAP_ZOOM = 8;

    // init map
    const map = L.map('map').setView(MAP_CENTER, MAP_ZOOM);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', { maxZoom: 19, attribution: '&copy; OpenStreetMap contributors' }).addTo(map);

    const greenIcon = L.icon({
      iconUrl: 'https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-2x-green.png',
      shadowUrl: 'https://unpkg.com/leaflet@1.9.4/dist/images/marker-shadow.png',
      iconSize: [25,41], iconAnchor: [12,41], popupAnchor: [1,-34], shadowSize: [41,41]
    });

    function loadMarkersFromStorage() { try { const raw = localStorage.getItem(STORAGE_KEY); return raw ? JSON.parse(raw) : []; } catch(e){ return []; } }
    function saveMarkersToStorage(list) { localStorage.setItem(STORAGE_KEY, JSON.stringify(list)); }
    function loadGeocodeCache() { try { return JSON.parse(localStorage.getItem(GEOCODE_CACHE_KEY) || '{}'); } catch (e) { return {}; } }
    function saveGeocodeCache(obj) { localStorage.setItem(GEOCODE_CACHE_KEY, JSON.stringify(obj)); }

    let markers = [];
    let markerData = loadMarkersFromStorage();

    function buildPopupHtml(d) {
      const fullnameLine = d.fullname ? `<div>ผู้ติดต่อ: ${d.fullname}</div>` : '';
      return `<div class="popup-card"><strong>${d.place}${d.province ? ' - ' + d.province : ''}</strong>${fullnameLine}<div style="margin-top:6px; text-align:right;"><button data-id="${d.id}" class="edit-marker inline-block text-xs px-2 py-1 rounded border">แก้ไข</button></div></div>`;
    }

    function clearAllMarkers() { markers.forEach(m=>map.removeLayer(m)); markers = []; }
    function renderAllMarkers() {
      clearAllMarkers();
      const bounds = [];
      markerData.forEach(d => {
        const m = L.marker([d.lat, d.lng], { icon: greenIcon }).addTo(map);
        m.bindPopup(buildPopupHtml(d)); m._markerId = d.id; m.on('popupopen', onPopupOpen); markers.push(m); bounds.push([d.lat, d.lng]);
      });
      if (bounds.length) { try { map.fitBounds(bounds, { padding: [40,40] }); } catch(e){} }
    }

    function onPopupOpen(e) {
      const container = e.popup.getElement(); if (!container) return; const btn = container.querySelector('.edit-marker'); if (btn) btn.addEventListener('click', ()=>{ const id = btn.getAttribute('data-id'); startEditingMarker(id); e.popup._close(); });
    }

    function addOrUpdateMarker(data) { if (!data.id) data.id = 'm_'+Date.now(); const idx = markerData.findIndex(x=>x.id===data.id); if (idx>=0) markerData[idx]=data; else markerData.push(data); saveMarkersToStorage(markerData); renderAllMarkers(); }

    function startEditingMarker(id) {
      const d = markerData.find(x=>x.id===id);
      if (!d) return;
      document.getElementById('editingId').value = d.id;
      document.getElementById('fullname').value = d.fullname || '';
      document.getElementById('province').value = d.province || '';
      document.getElementById('lat').value = d.lat;
      document.getElementById('lng').value = d.lng;
      window.scrollTo({ top:0, behavior:'smooth' });
    }

    async function geocode(query) {
      if (!query) return null; const cache = loadGeocodeCache(); if (cache[query]) return cache[query]; const url = `https://nominatim.openstreetmap.org/search?format=json&limit=1&q=${encodeURIComponent(query)}&email=${encodeURIComponent(NOMINATIM_EMAIL)}`;
      try { const resp = await fetch(url, { headers: { 'Accept-Language': 'th' } }); if (!resp.ok) return null; const arr = await resp.json(); if (arr && arr.length>0) { const r = { lat: parseFloat(arr[0].lat), lon: parseFloat(arr[0].lon) }; cache[query]=r; saveGeocodeCache(cache); return r; } } catch(e){ console.warn('Geocode error', e); } return null;
    }

    // form handlers
    const form = document.getElementById('markerForm');
    form.addEventListener('submit', async (ev)=>{
      ev.preventDefault();
      const editingId = document.getElementById('editingId').value || null;
      const place = document.getElementById('place').value.trim();
      const fullname = (document.getElementById('fullname') && document.getElementById('fullname').value) ? document.getElementById('fullname').value.trim() : '';
      const province = document.getElementById('province').value.trim();
      let lat = document.getElementById('lat').value.trim();
      let lng = document.getElementById('lng').value.trim();

      if (!place && !province) { alert('กรุณากรอกชื่อสถานที่หรือจังหวัด'); return; }

      if (!lat || !lng) {
        const q = [place, province].filter(Boolean).join(', ');
        if (!q) { alert('กรุณากรอกชื่อสถานที่หรือพิกัด'); return; }
        const g = await geocode(q);
        if (!g) { alert('ไม่พบพิกัดสำหรับคำค้นนี้'); return; }
        lat = g.lat; lng = g.lon;
      } else {
        lat = parseFloat(lat); lng = parseFloat(lng);
      }

      const data = {
        id: editingId || null,
        place: place || province,
        fullname: fullname || null,
        province: province,
        lat: Number(lat),
        lng: Number(lng)
      };

      addOrUpdateMarker(data);
      document.getElementById('editingId').value = '';
      form.reset();
    });
    document.getElementById('clearBtn').addEventListener('click', ()=> form.reset());

    // ensure example
    (function ensureExample(){ if (!markerData || markerData.length===0) { const sample = { id:'chiangmai_1', place:'เชียงใหม่', fullname: 'นายตัวอย่าง', province:'เชียงใหม่', lat:18.7883, lng:98.9853 }; markerData=[sample]; saveMarkersToStorage(markerData); } })();

    renderAllMarkers();
    window.addEventListener('resize', ()=>{ const b = markerData.map(d=>[d.lat,d.lng]); if (b.length) map.fitBounds(b, { padding:[40,40] }); });

    // expose helpers
    window.longan = { getData: ()=>markerData, clearAll: ()=>{ markerData=[]; saveMarkersToStorage(markerData); renderAllMarkers(); }, removeById: (id)=>{ markerData = markerData.filter(x=>x.id!==id); saveMarkersToStorage(markerData); renderAllMarkers(); } };

  } catch (e) { console.error('Init error', e); }
});
</script>

<style scoped>
.leaflet-popup-content-wrapper { max-width: 260px; }
</style>
