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
  }
  #mode-bar {
    background: #fff8e1;
    padding: 8px 20px;
    text-align: center;
    font-size: 13px;
    font-weight: bold;
    color: #e65100;
    border-top: 1px solid #ffe082;
    flex-shrink: 0;
    min-height: 36px;
  }
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
  #location-value {
    font-size: 22px;
    font-weight: bold;
    color: #2979ff;
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

<div id="mode-bar">🗺️ 지도를 탭하면 마커가 변신해요!</div>

<div id="map-wrap">
  <div id="map"></div>
</div>

<div id="bottom">
  <div id="location-row">
    <span id="location-icon">📍</span>
    <span id="location-label" style="font-size:16px;color:#555;">목적지:</span>
    <span id="location-value">---</span>
  </div>
  <button id="copy-btn" onclick="copyMessage()">
    <span id="btn-icon">📋</span>
    <span id="btn-text">위치 메시지 복사 (카톡 전송용)</span>
  </button>
  <div id="guide-text">📱 아이폰: 버튼 누른 후 카카오톡 → 채팅창 길게 누르기 → 붙여넣기</div>
</div>

<script>
  const animals = [
    { emoji: '📍', name: '일반',   msg: null },
    { emoji: '🐶', name: '강아지', msg: '멍멍! 여기 주차하면 꼬리칠게요!' },
    { emoji: '🐱', name: '고양이', msg: '냥~ 여기 내 구역이에요.' },
    { emoji: '🐷', name: '돼지',   msg: '꿀꿀~ 맛있는 자리 찾았다!' },
    { emoji: '🐮', name: '소',     msg: '음메~ 이 넓은 자리 딱이네요.' },
    { emoji: '🐻', name: '곰',     msg: '으르렁! 곰이 점령한 구역입니다.' },
    { emoji: '🐰', name: '토끼',   msg: '깡충깡충! 여기까지 뛰어왔어요.' },
    { emoji: '🐔', name: '닭',     msg: '꼬끼오~ 새벽부터 자리 맡았어요.' },
    { emoji: '🦤', name: '타조',   msg: '빠른 타조도 이 자리는 못 지나쳐요.' },
    { emoji: '🐐', name: '염소',   msg: '메에~ 험한 길도 여기까지 왔어요.' },
  ];

  let modeIndex = 0;
  let selectedLocation = '---';
  let markers = [];

  const map = L.map('map').setView([37.4763, 126.5233], 17);
  L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);

  const locations = [
    {name:"지하주차장", pos:[37.476878, 126.523374]},
    {name:"주차장 입구 A공터",     pos:[37.476729, 126.522995]},
    {name:"퇴근시 나가는 쪽 길가 B공터",     pos:[37.476613, 126.524049]},
    {name:"남쪽 C공터",     pos:[37.47615,  126.523360]},
    {name:"공사장 입구 D공터",     pos:[37.476385, 126.522816]},
    {name:"남쪽 E공터",     pos:[37.476242, 126.523775]},
    {name:"남쪽 F공터",     pos:[37.476059, 126.523025]},
    {name:"공사장 G공터",     pos:[37.476638, 126.522514]},
    {name:"남쪽 H공터",     pos:[37.47589,  126.523507]},
    {name:"스타리아 주로 대는 I공터",     pos:[37.475827, 126.523229]},
    {name:"가장 먼 남쪽 J공터",     pos:[37.475652, 126.523681]}
  ];

  function makeEmojiIcon(emoji) {
    const canvas = document.createElement('canvas');
    canvas.width = 50; canvas.height = 50;
    const ctx = canvas.getContext('2d');
    ctx.font = '38px serif';
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillText(emoji, 25, 25);
    return L.icon({
      iconUrl: canvas.toDataURL(),
      iconSize: [45, 45],
      iconAnchor: [22, 45],
      popupAnchor: [0, -45]
    });
  }

  // 마커 생성
  locations.forEach(loc => {
    const marker = L.marker(loc.pos).addTo(map);
    marker.bindTooltip(loc.name, {sticky: true, direction: 'top'});
    marker.on('click', e => {
      L.DomEvent.stopPropagation(e);
      selectedLocation = loc.name;
      document.getElementById('location-value').textContent = loc.name;
      document.getElementById('copy-btn').classList.remove('copied');
      document.getElementById('btn-icon').textContent = '📋';
      document.getElementById('btn-text').textContent = '위치 메시지 복사 (카톡 전송용)';

      // 동물 모드면 팝업 표시
      const a = animals[modeIndex];
      if (a.msg) {
        const hours = Math.floor(Math.random() * 5) + 1;
        marker.bindPopup(
          `${a.emoji} <b>${loc.name}</b><br>주차 ${hours}시간 경과<br><small>${a.msg}</small>`
        ).openPopup();
      }
    });
    markers.push(marker);
  });

  // 지도 탭 → 다음 동물 모드로
  map.on('click', () => {
    modeIndex = (modeIndex + 1) % animals.length;
    const a = animals[modeIndex];

    // 모드바 업데이트
    document.getElementById('mode-bar').textContent =
      modeIndex === 0 ? '🗺️ 일반 모드' : `${a.emoji} ${a.name} 모드 활성화!`;

    // 마커 아이콘 업데이트
    markers.forEach(m => {
      m.setIcon(modeIndex === 0 ? new L.Icon.Default() : makeEmojiIcon(a.emoji));
      if (m.getPopup()) m.closePopup();
    });
  });

  function copyMessage() {
    if (selectedLocation === '---') {
      alert('지도에서 위치를 먼저 선택해주세요.');
      return;
    }
    const car = document.getElementById('car-select').value;
    const a = animals[modeIndex];
    const animalTag = a.msg ? ` ${a.emoji}` : '';
    const msg = `[${car}] 주차 위치: ${selectedLocation}${animalTag}\n다음 운전자는 이곳으로 이동 바랍니다.`;

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

  setTimeout(() => map.invalidateSize(), 300);
</script>
</body>
</html>
