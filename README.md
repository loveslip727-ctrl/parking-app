<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
<title>주차 관리 시스템</title>
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/leaflet@1.9.3/dist/leaflet.css"/>
<script src="https://cdn.jsdelivr.net/npm/leaflet@1.9.3/dist/leaflet.js"></script>
<style>
  * { box-sizing: border-box; margin: 0; padding: 0; }

  body {
    font-family: 'Apple SD Gothic Neo', 'Malgun Gothic', sans-serif;
    background: #f5f6fa;
    height: 100dvh;
    display: flex;
    flex-direction: column;
    overflow: hidden;
  }

  #appbar {
    background: #2979ff;
    color: white;
    text-align: center;
    padding: 14px;
    font-size: 18px;
    font-weight: bold;
    flex-shrink: 0;
    transition: background 0.3s;
  }

  #car-row {
    background: white;
    padding: 10px 20px;
    display: flex;
    align-items: center;
    gap: 12px;
    box-shadow: 0 2px 4px rgba(0,0,0,0.08);
    flex-shrink: 0;
  }
  #car-row label { font-weight: bold; font-size: 14px; }
  #car-select {
    padding: 6px 12px;
    border: 1.5px solid #2979ff;
    border-radius: 8px;
    font-size: 15px;
    background: white;
    color: #222;
    cursor: pointer;
    transition: border-color 0.3s;
  }

  #toggle-row {
    background: white;
    padding: 10px 20px;
    display: flex;
    align-items: center;
    gap: 14px;
    border-top: 1px solid #eee;
    flex-shrink: 0;
  }
  #toggle-label { font-size: 14px; font-weight: bold; flex: 1; }
  #toggle-sub { font-size: 12px; color: #888; }

  .toggle-switch {
    position: relative;
    width: 50px;
    height: 28px;
    flex-shrink: 0;
  }
  .toggle-switch input { opacity: 0; width: 0; height: 0; }
  .slider {
    position: absolute; inset: 0;
    background: #ccc;
    border-radius: 28px;
    cursor: pointer;
    transition: background 0.3s;
  }
  .slider:before {
    content: '';
    position: absolute;
    width: 22px; height: 22px;
    left: 3px; top: 3px;
    background: white;
    border-radius: 50%;
    transition: transform 0.3s;
  }
  input:checked + .slider { background: #ff6f00; }
  input:checked + .slider:before { transform: translateX(22px); }

  #map-wrap {
    flex: 1;
    padding: 10px;
    min-height: 0;
  }
  #map {
    width: 100%;
    height: 100%;
    border-radius: 14px;
    border: 2px solid #2979ff;
    transition: border-color 0.3s;
  }

  #bottom {
    background: white;
    padding: 16px 20px 20px;
    flex-shrink: 0;
    box-shadow: 0 -2px 8px rgba(0,0,0,0.08);
  }
  #location-row {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
    margin-bottom: 14px;
  }
  #location-icon { font-size: 22px; }
  #location-label { font-size: 16px; color: #555; }
  #location-value {
    font-size: 22px;
    font-weight: bold;
    color: #2979ff;
    transition: color 0.3s;
  }

  #copy-btn {
    width: 100%;
    padding: 16px;
    background: #fdd835;
    border: none;
    border-radius: 12px;
    font-size: 16px;
    font-weight: bold;
    color: #222;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
    transition: background 0.3s, transform 0.1s;
    -webkit-tap-highlight-color: transparent;
  }
  #copy-btn:active { transform: scale(0.97); }
  #copy-btn.copied { background: #43a047; color: white; }

  #guide-text {
    text-align: center;
    font-size: 11px;
    color: #aaa;
    margin-top: 8px;
  }

  body.nureongi #appbar { background: #e65100; }
  body.nureongi #car-select { border-color: #e65100; }
  body.nureongi #map { border-color: #e65100; }
  body.nureongi #location-value { color: #e65100; }
</style>
</head>
<body>

<div id="appbar">🚗 주차 관리 시스템</div>

<div id="car-row">
  <span>🚐</span>
  <label for="car-select">차종 선택:</label>
  <select id="car-select">
    <option value="스타리아">스타리아</option>
    <option value="캐스퍼">캐스퍼</option>
  </select>
</div>

<div id="toggle-row">
  <div>
    <div id="toggle-label">📍 일반 지도 모드</div>
    <div id="toggle-sub">표준 마커로 위치를 표시합니다.</div>
  </div>
  <label class="toggle-switch">
    <input type="checkbox" id="mode-toggle">
    <span class="slider"></span>
  </label>
</div>

<div id="map-wrap">
  <div id="map"></div>
</div>

<div id="bottom">
  <div id="location-row">
    <span id="location-icon">📍</span>
    <span id="location-label">목적지:</span>
    <span id="location-value">---</span>
  </div>
  <button id="copy-btn" onclick="copyMessage()">
    <span id="btn-icon">📋</span>
    <span id="btn-text">위치 메시지 복사 (카톡 전송용)</span>
  </button>
  <div id="guide-text">📱 아이폰: 버튼 누른 후 카카오톡 → 채팅창 길게 누르기 → 붙여넣기</div>
</div>

<script>
  const DOG_ICON_URL = 'https://cdn-icons-png.flaticon.com/512/616/616408.png';

  // 지하주차장 기준 직선거리 가까운 순 정렬 / 이름 A공터 형식
  const locations = [
    {name:"지하주차장", pos:[37.476878, 126.523374]},
    {name:"A공터",     pos:[37.476729, 126.522995]}, // 약 45m
    {name:"B공터",     pos:[37.476613, 126.524049]}, // 약 80m
    {name:"C공터",     pos:[37.47615, 126.523360]},  // 약 81m
    {name:"D공터",     pos:[37.476385, 126.522816]}, // 약 83m
    {name:"E공터",     pos:[37.476242, 126.523775]}, // 약 83m
    {name:"F공터",     pos:[37.476059, 126.523025]}, // 약 99m
    {name:"G공터",     pos:[37.476638, 126.522514]}, // 약 99m
    {name:"H공터",     pos:[37.47589,  126.523507]}, // 약 111m
    {name:"I공터",     pos:[37.475827, 126.523229]}, // 약 122m (교차로에서 5m 이동)
    {name:"J공터",     pos:[37.475652, 126.523681]}  // 약 140m
  ];

  let selectedLocation = '---';
  let isNureongi = false;
  let isDogMode = false;
  let markers = [];

  const map = L.map('map').setView([37.4763, 126.5233], 17);
  L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);

  const dogIcon = L.icon({
    iconUrl: DOG_ICON_URL,
    iconSize: [45, 45],
    iconAnchor: [22, 45],
    popupAnchor: [0, -45]
  });

  function addMarkers(nureongi) {
    markers.forEach(m => map.removeLayer(m));
    markers = [];
    locations.forEach(loc => {
      const marker = L.marker(loc.pos, nureongi ? {icon: dogIcon} : {}).addTo(map);
      marker.bindTooltip(loc.name, {sticky: true, direction: 'top'});
      marker.on('click', e => {
        L.DomEvent.stopPropagation(e);
        selectLocation(loc.name);
      });
      markers.push(marker);
    });
  }

  // 지도 아무데나 탭 → 개 마커 토글
  map.on('click', function() {
    isDogMode = !isDogMode;
    markers.forEach(m => {
      m.setIcon(isDogMode ? dogIcon : new L.Icon.Default());
    });
  });

  function selectLocation(name) {
    selectedLocation = name;
    document.getElementById('location-value').textContent = name;
    const btn = document.getElementById('copy-btn');
    btn.classList.remove('copied');
    document.getElementById('btn-icon').textContent = '📋';
    document.getElementById('btn-text').textContent = '위치 메시지 복사 (카톡 전송용)';
  }

  // 모드 토글 (토글 스위치 - 기존 기능 유지)
  document.getElementById('mode-toggle').addEventListener('change', function() {
    isNureongi = this.checked;
    document.body.classList.toggle('nureongi', isNureongi);
    document.getElementById('toggle-label').textContent = isNureongi ? '🐕 안내 모드 활성화' : '📍 일반 지도 모드';
    document.getElementById('toggle-sub').textContent = isNureongi ? '강아지 마커가 명당을 안내합니다.' : '표준 마커로 위치를 표시합니다.';
    document.getElementById('location-icon').textContent = isNureongi ? '🐾' : '📍';
    isDogMode = false; // 토글 바꾸면 탭 토글 초기화
    addMarkers(isNureongi);
  });

  function copyMessage() {
    if (selectedLocation === '---') {
      alert('지도에서 위치를 먼저 선택해주세요.');
      return;
    }
    const car = document.getElementById('car-select').value;
    const msg = isNureongi
      ? `[${car}] 주차 안내: ${selectedLocation} (명당🐕)`
      : `[${car}] 주차 위치: ${selectedLocation} (확인 요망)`;

    const input = document.createElement('input');
    input.setAttribute('type', 'text');
    input.setAttribute('readonly', '');
    input.value = msg;
    input.style.cssText = 'position:fixed;top:0;left:0;width:2em;height:2em;padding:0;border:none;outline:none;box-shadow:none;background:transparent;font-size:16px;opacity:0;';
    document.body.appendChild(input);
    input.focus();
    input.setSelectionRange(0, 9999);

    let success = false;
    try { success = document.execCommand('copy'); } catch(e) {}
    document.body.removeChild(input);

    if (!success && navigator.clipboard && navigator.clipboard.writeText) {
      navigator.clipboard.writeText(msg).then(showCopied).catch(() => {
        alert('복사 실패. 직접 복사해주세요:\n' + msg);
      });
      return;
    }
    success ? showCopied() : alert('복사 실패. 직접 복사해주세요:\n' + msg);
  }

  function showCopied() {
    const btn = document.getElementById('copy-btn');
    btn.classList.add('copied');
    document.getElementById('btn-icon').textContent = '✅';
    document.getElementById('btn-text').textContent = '복사 완료! → 카톡에 붙여넣기 하세요';
  }

  addMarkers(false);
  setTimeout(() => map.invalidateSize(), 300);
</script>
</body>
</html>
