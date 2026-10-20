# myctugo
<!DOCTYPE html>
<html lang="vi">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Bản đồ Campus Trường Đại học</title>
  
  <!-- Font chữ Google Roboto & Leaflet CSS -->
  <link href="https://fonts.googleapis.com/css2?family=Roboto:wght@400;500;700&display=swap" rel="stylesheet">
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>

  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; font-family: 'Roboto', sans-serif; }
    #map { width: 100vw; height: 100vh; z-index: 1; }

    /* Thanh tìm kiếm nổi lên trên bản đồ */
    .search-box {
      position: absolute; top: 15px; left: 50%; transform: translateX(-50%);
      z-index: 1000; width: 90%; max-width: 400px;
      background: white; border-radius: 30px;
      padding: 8px 16px; box-shadow: 0 4px 15px rgba(0,0,0,0.15);
      display: flex; align-items: center;
    }
    .search-box input {
      width: 100%; border: none; outline: none; font-size: 14px; padding: 6px;
    }

    /* Thiết kế lại popup thông tin tòa nhà */
    .leaflet-popup-content-wrapper {
      border-radius: 12px; padding: 0; overflow: hidden;
      box-shadow: 0 6px 20px rgba(0,0,0,0.15);
    }
    .leaflet-popup-content { margin: 0 !important; width: 230px !important; }
    
    .card-img { width: 100%; height: 130px; object-fit: cover; }
    .card-body { padding: 12px; }
    .card-title { font-size: 15px; font-weight: 700; color: #1e293b; margin-bottom: 4px; }
    .card-desc { font-size: 12px; color: #64748b; line-height: 1.4; }

    /* Nút định vị GPS nổi */
    .btn-locate {
      position: absolute; bottom: 30px; right: 20px; z-index: 1000;
      background: #2563eb; color: white; border: none;
      width: 50px; height: 50px; border-radius: 50%;
      box-shadow: 0 4px 12px rgba(37, 99, 235, 0.4);
      cursor: pointer; font-size: 20px;
      display: flex; align-items: center; justify-content: center;
      transition: transform 0.2s;
    }
    .btn-locate:active { transform: scale(0.9); }

    /* Hiệu ứng vị trí người dùng */
    .user-dot {
      width: 16px; height: 16px; background-color: #2563eb;
      border: 3px solid #ffffff; border-radius: 50%;
      box-shadow: 0 0 0 rgba(37, 99, 235, 0.4);
      animation: pulse 1.8s infinite;
    }
    @keyframes pulse {
      0% { box-shadow: 0 0 0 0 rgba(37, 99, 235, 0.7); }
      70% { box-shadow: 0 0 0 15px rgba(37, 99, 235, 0); }
      100% { box-shadow: 0 0 0 0 rgba(37, 99, 235, 0); }
    }
  </style>
</head>
<body>

  <!-- Ô tìm kiếm -->
  <div class="search-box">
    <input type="text" id="searchInput" placeholder="🔍 Tìm tòa nhà, phòng học..." onkeyup="searchBuilding()">
  </div>

  <!-- Nút định vị -->
  <button class="btn-locate" onclick="centerUser()">🎯</button>

  <div id="map"></div>

  <script>
    // 1. Khởi tạo bản đồ
    const map = L.map('map', { zoomControl: false }).setView([10.762622, 106.682229], 17);

    // 2. Dùng nền bản đồ hiện đại (CartoDB Voyager)
    L.tileLayer('https://{s}.basemaps.cartocdn.com/rastertiles/voyager/{z}/{x}/{y}{r}.png', {
      maxZoom: 19,
      attribution: '© OpenStreetMap'
    }).addTo(map);

    // 3. Dữ liệu mẫu tòa nhà
    const buildings = [
      { name: "Nhà A1 - Giảng đường", lat: 10.762900, lng: 106.682100, img: "https://via.placeholder.com/230x130?text=Nha+A1", desc: "Giảng đường chính, phòng A101-A505." },
      { name: "Thư viện Center", lat: 10.762300, lng: 106.682500, img: "https://via.placeholder.com/230x130?text=Thu+Vien", desc: "Không gian học tập & máy tính." }
    ];

    const markers = [];

    // 4. Render các marker
    buildings.forEach(item => {
      const popup = `
        <div>
          <img src="${item.img}" class="card-img"/>
          <div class="card-body">
            <div class="card-title">${item.name}</div>
            <div class="card-desc">${item.desc}</div>
          </div>
        </div>
      `;
      const marker = L.marker([item.lat, item.lng]).addTo(map).bindPopup(popup);
      markers.push({ name: item.name.toLowerCase(), marker: marker, lat: item.lat, lng: item.lng });
    });

    // 5. Định vị GPS
    let userMarker = null, userLat = null, userLng = null;
    const userIcon = L.divIcon({ className: 'user-dot', iconSize: [16, 16] });

    if ("geolocation" in navigator) {
      navigator.geolocation.watchPosition(pos => {
        userLat = pos.coords.latitude;
        userLng = pos.coords.longitude;
        if (userMarker) {
          userMarker.setLatLng([userLat, userLng]);
        } else {
          userMarker = L.marker([userLat, userLng], { icon: userIcon }).addTo(map);
        }
      }, null, { enableHighAccuracy: true });
    }

    function centerUser() {
      if (userLat && userLng) map.setView([userLat, userLng], 18);
      else alert("Đang định vị GPS, vui lòng đợi vài giây...");
    }

    // 6. Tính năng tìm kiếm cơ bản
    function searchBuilding() {
      const query = document.getElementById('searchInput').value.toLowerCase();
      const match = markers.find(m => m.name.includes(query));
      if (match && query.length > 2) {
        map.setView([match.lat, match.lng], 18);
        match.marker.openPopup();
      }
    }
  </script>
</body>
</html>
