<template>
  <div class="page-shell" :class="isDarkMode ? 'theme-dark' : 'theme-light'">
    <div v-if="!mapReady && !mapError" class="loading-overlay">
      <div class="loading-card">
        <p class="loading-title">카카오맵을 먼저 불러오는 중입니다.</p>
        <p class="loading-subtitle">지도가 준비되면 기능 패널이 나타납니다.</p>
      </div>
    </div>

    <div v-if="mapError" class="error-overlay">
      <div class="loading-card">
        <p class="loading-title">지도를 불러오지 못했습니다.</p>
        <p class="loading-subtitle">{{ mapError }}</p>
      </div>
    </div>

    <div v-if="mapReady" class="toolbar">
      <div class="toolbar-inner">
        <div class="toolbar-top">
          <p class="toolbar-title">카카오맵이 먼저 보이고, 기능은 그 다음에 표시됩니다.</p>
          <button class="theme-toggle" @click="toggleTheme">{{ isDarkMode ? '☀️ 라이트' : '🌙 다크' }}</button>
        </div>
        <div class="toolbar-meta">
          <span class="count-pill">선택된 카테고리 수: {{ activeCategories.length }}</span>
        </div>
        <div class="category-list">
          <button
            v-for="c in categories"
            :key="c"
            class="category-btn"
            :class="{ active: activeCategories.includes(c) }"
            @click="toggleCategory(c)"
          >
            <span class="emoji">✨</span>
            <span>{{ c }}</span>
          </button>
        </div>
      </div>
    </div>

    <div id="map" class="map-area"></div>

    <div v-if="selectedPlace" class="detail-panel" :style="detailPanelStyle">
      <button class="detail-close" @click="closeDetailPanel">✕</button>
      <div class="detail-content">
        <div class="detail-header">
          <span class="detail-badge">📍 상세 정보</span>
          <h3>{{ selectedPlace.title }}</h3>
        </div>
        <p v-if="selectedPlace.description" class="detail-description">{{ selectedPlace.description }}</p>
        <p v-if="selectedPlace.address" class="detail-address">📌 {{ selectedPlace.address }}</p>
        <img
          v-if="selectedPlace.image"
          :src="selectedPlace.image"
          :alt="selectedPlace.title"
          class="detail-image"
          @error="handleImageError"
        />
        <div v-else class="detail-placeholder">📷 이미지가 없어요</div>
      </div>
    </div>
  </div>
</template>

<script>
import tourismData from '../서울_관광지.json';
import leisureData from '../서울_레포츠.json';
import cultureData from '../서울_문화시설.json';
import shoppingData from '../서울_쇼핑.json';
import lodgingData from '../서울_숙박.json';
import courseData from '../서울_여행코스.json';
import festivalData from '../서울_축제공연행사.json';

export default {
  data() {
    return {
      map: null,
      regionMarkers: [],
      poiMarkersByCategory: {},
      customOverlay: null,
      selectedPlace: null,
      detailPanelStyle: {
        left: '16px',
        top: '16px'
      },
      isDarkMode: true,
      categories: ['관광지', '레포츠', '문화시설', '쇼핑', '숙박', '여행코스', '축제공연행사'],
      activeCategories: [],
      categoryColors: {
        관광지: '#ff6b6b',
        레포츠: '#4ecdc4',
        문화시설: '#ffd166',
        쇼핑: '#a06cd5',
        숙박: '#06d6a0',
        여행코스: '#118ab2',
        축제공연행사: '#ef476f'
      },
      mapReady: false,
      mapError: '',
      seoulDataByCategory: {
        관광지: tourismData.items || [],
        레포츠: leisureData.items || [],
        문화시설: cultureData.items || [],
        쇼핑: shoppingData.items || [],
        숙박: lodgingData.items || [],
        여행코스: courseData.items || [],
        축제공연행사: festivalData.items || []
      },
      seoulLoaded: false
    };
  },
  watch: {
    activeCategories() {
      this.updatePoiVisibility();
    }
  },
  mounted() {
    this.initTheme();
    this.loadKakaoMap();
  },
  beforeUnmount() {
    if (this.customOverlay) {
      this.customOverlay.setMap(null);
    }
    this.regionMarkers.forEach(marker => marker.setMap(null));
    this.regionMarkers = [];
    Object.values(this.poiMarkersByCategory).forEach(items => {
      items.forEach(item => {
        item.marker.setMap(null);
        item.overlay.setMap(null);
      });
    });
    this.poiMarkersByCategory = {};
  },
  methods: {
    createCustomMarkerImage(label, color) {
      const svg = `
        <svg xmlns="http://www.w3.org/2000/svg" width="60" height="60" viewBox="0 0 60 60">
          <path d="M30 4c-10.5 0-19 8.5-19 19 0 13.3 19 31 19 31s19-17.7 19-31c0-10.5-8.5-19-19-19z" fill="${color}"/>
          <circle cx="30" cy="22" r="8" fill="#fff"/>
          <text x="30" y="26" text-anchor="middle" font-size="12">${label}</text>
        </svg>`;

      return new window.kakao.maps.MarkerImage(
        'data:image/svg+xml;charset=UTF-8,' + encodeURIComponent(svg),
        new window.kakao.maps.Size(44, 44),
        { offset: new window.kakao.maps.Point(22, 44) }
      );
    },
    createBubbleContent(title, description) {
      return `
        <div class="cute-bubble">
          <div class="cute-bubble__title">${title}</div>
          <div class="cute-bubble__desc">${description}</div>
        </div>
      `;
    },
    initTheme() {
      try {
        const saved = localStorage.getItem('localhub-theme');
        if (saved === 'dark' || saved === 'light') {
          this.isDarkMode = saved === 'dark';
          return;
        }
      } catch (error) {
        console.warn('theme storage error', error);
      }
      this.isDarkMode = window.matchMedia && window.matchMedia('(prefers-color-scheme: dark)').matches;
    },
    toggleTheme() {
      this.isDarkMode = !this.isDarkMode;
      try {
        localStorage.setItem('localhub-theme', this.isDarkMode ? 'dark' : 'light');
      } catch (error) {
        console.warn('theme storage write error', error);
      }
    },
    toggleCategory(category) {
      if (this.activeCategories.includes(category)) {
        this.activeCategories = this.activeCategories.filter(item => item !== category);
      } else {
        this.activeCategories = [...this.activeCategories, category];
      }
    },
    closeDetailPanel() {
      this.selectedPlace = null;
      this.detailPanelStyle = { left: '16px', top: '16px' };
    },
    positionDetailPanel(position) {
      if (!position || !this.map) {
        this.detailPanelStyle = { left: '16px', top: '16px' };
        return;
      }

      const mapContainer = document.getElementById('map');
      const panel = this.$el?.querySelector('.detail-panel');
      if (!mapContainer || !panel) {
        return;
      }

      const projection = this.map.getProjection();
      if (!projection) {
        return;
      }

      const point = projection.pointFromCoords(position);
      const pageRect = this.$el.getBoundingClientRect();
      const mapRect = mapContainer.getBoundingClientRect();
      const panelWidth = panel.offsetWidth || 260;
      const panelHeight = panel.offsetHeight || 220;

      let left = point.x + mapRect.left - pageRect.left + 16;
      let top = point.y + mapRect.top - pageRect.top + 16;

      const maxLeft = Math.max(16, mapRect.width - panelWidth - 16);
      const maxTop = Math.max(16, mapRect.height - panelHeight - 16);

      left = Math.min(Math.max(left, 16), maxLeft);
      top = Math.min(Math.max(top, 16), maxTop);

      this.detailPanelStyle = { left: `${left}px`, top: `${top}px` };
    },
    createFallbackImage(label) {
      const safeLabel = String(label || '장소').replace(/&/g, '&amp;');
      const svg = `
        <svg xmlns="http://www.w3.org/2000/svg" width="400" height="240" viewBox="0 0 400 240">
          <rect width="400" height="240" rx="24" fill="#fff3f7" />
          <circle cx="200" cy="100" r="56" fill="#ffd6e4" />
          <path d="M145 185c10-36 45-56 55-56s45 20 55 56" fill="#ff8fab" />
          <text x="200" y="210" text-anchor="middle" font-size="20" fill="#a24a67" font-family="Arial, sans-serif">${safeLabel}</text>
        </svg>`;
      return 'data:image/svg+xml;charset=UTF-8,' + encodeURIComponent(svg);
    },
    handleImageError(event) {
      if (!this.selectedPlace) return;
      const fallback = this.createFallbackImage(this.selectedPlace.title || '장소');
      if (event.target.src !== fallback) {
        event.target.src = fallback;
        this.selectedPlace.image = fallback;
      }
    },
    getPlaceAddress(item) {
      return [item.addr1, item.addr2].filter(Boolean).join(' ');
    },
    getPlaceImage(item) {
      const rawImage = item.firstimage || item.firstimage2 || '';
      return rawImage && String(rawImage).trim() ? String(rawImage).trim() : this.createFallbackImage(item.title || '장소');
    },
    showBubble(marker, title, description, address = '', image = '', position = null) {
      if (this.customOverlay) {
        this.customOverlay.setMap(null);
      }
      this.customOverlay = null;
      this.selectedPlace = { title, description, address, image };
      this.$nextTick(() => {
        this.positionDetailPanel(position || marker?.getPosition?.());
      });
    },
    loadKakaoMap() {
      if (window.kakao?.maps) {
        this.initMap();
        return;
      }

      const appKey = import.meta.env.VITE_KAKAO_MAP_KEY || 'YOUR_APP_KEY';
      if (!appKey || appKey === 'YOUR_APP_KEY') {
        this.mapError = '카카오 앱 키가 아직 설정되지 않았습니다. .env 파일에 VITE_KAKAO_MAP_KEY를 넣어주세요.';
        return;
      }

      const existingScript = document.querySelector('script[data-kakao-map]');
      if (existingScript) {
        existingScript.addEventListener('load', () => {
          window.kakao.maps.load(() => this.initMap());
        }, { once: true });
        return;
      }

      const script = document.createElement('script');
      script.setAttribute('data-kakao-map', 'true');
      script.src = `https://dapi.kakao.com/v2/maps/sdk.js?appkey=${appKey}&libraries=services&autoload=false`;
      script.async = true;
      script.onload = () => {
        window.kakao.maps.load(() => this.initMap());
      };
      script.onerror = () => {
        this.mapError = '카카오 지도 스크립트를 불러오지 못했습니다. 앱 키와 도메인을 확인해주세요.';
      };
      document.head.appendChild(script);
    },
    initMap() {
      const container = document.getElementById('map');
      if (!container) return;

      const options = {
        center: new window.kakao.maps.LatLng(36.5, 127.8),
        level: 7
      };
      this.map = new window.kakao.maps.Map(container, options);
      this.mapReady = true;

      const regions = [
        { name: '서울', lat: 37.5665, lng: 126.9780, level: 6 },
        { name: '대전', lat: 36.3504, lng: 127.3845, level: 8 },
        { name: '구미', lat: 36.1190, lng: 128.3440, level: 8 },
        { name: '부산', lat: 35.1796, lng: 129.0756, level: 7 },
        { name: '광주', lat: 35.1595, lng: 126.8526, level: 7 }
      ];

      regions.forEach(r => {
        const marker = new window.kakao.maps.Marker({
          map: this.map,
          position: new window.kakao.maps.LatLng(r.lat, r.lng),
          title: r.name,
          image: this.createCustomMarkerImage('📍', r.name === '서울' ? '#ff6b6b' : '#6ec5ff')
        });
        window.kakao.maps.event.addListener(marker, 'click', () => {
          this.map.setLevel(r.level);
          this.map.setCenter(new window.kakao.maps.LatLng(r.lat, r.lng));
          if (r.name === '서울') {
            this.loadSeoulPois();
          }
          this.showBubble(
            marker,
            r.name,
            r.name === '서울' ? '서울 여행지 데이터를 불러왔어요.' : `${r.name} 지역의 여행 정보를 확인해보세요.`,
            '',
            '',
            marker.getPosition()
          );
        });
        this.regionMarkers.push(marker);
      });
    },
    loadSeoulPois() {
      if (this.seoulLoaded) {
        this.updatePoiVisibility();
        return;
      }

      Object.entries(this.seoulDataByCategory).forEach(([category, items]) => {
        const categoryMarkers = [];
        items.forEach(item => {
          const lat = Number(item.mapy);
          const lng = Number(item.mapx);
          const position = new window.kakao.maps.LatLng(lat, lng);
          const marker = new window.kakao.maps.Marker({
            position,
            image: this.createCustomMarkerImage('📍', this.categoryColors[category] || '#ff8fab')
          });
          const overlay = new window.kakao.maps.CustomOverlay({
            position,
            content: this.createBubbleContent(item.title, item.addr1 || '서울 여행지예요.'),
            yAnchor: 1.25
          });
          window.kakao.maps.event.addListener(marker, 'click', () => {
            const address = this.getPlaceAddress(item);
            const image = this.getPlaceImage(item);
            const description = item.overview || (address ? '서울의 추천 여행지예요.' : '서울 여행지예요.');
            this.showBubble(marker, item.title, description, address, image, marker.getPosition());
          });
          categoryMarkers.push({ marker, overlay });
          marker.setMap(this.activeCategories.includes(category) ? this.map : null);
          overlay.setMap(null);
        });
        this.poiMarkersByCategory[category] = categoryMarkers;
      });

      this.seoulLoaded = true;
      this.updatePoiVisibility();
    },
    updatePoiVisibility() {
      for (const [cat, items] of Object.entries(this.poiMarkersByCategory)) {
        const show = this.activeCategories.includes(cat);
        items.forEach(item => {
          item.marker.setMap(show ? this.map : null);
          if (!show) {
            item.overlay.setMap(null);
          }
        });
      }

      const allMarkers = Object.values(this.poiMarkersByCategory).flat();
      allMarkers.forEach(item => {
        if (!item.marker || !this.map) return;
        const category = Object.keys(this.poiMarkersByCategory).find(key => this.poiMarkersByCategory[key].includes(item));
        const shouldDim = category && !this.activeCategories.includes(category);
        item.marker.setOpacity(shouldDim ? 0.3 : 1);
      });
    }
  }
};
</script>

<style scoped>
.page-shell {
  height: 100vh;
  display: flex;
  flex-direction: column;
  position: relative;
  overflow: hidden;
  background: var(--app-bg);
  color: var(--text-color);
  transition: background 0.25s ease, color 0.25s ease;
}

.page-shell.theme-light {
  --app-bg: #fffdfd;
  --text-color: #222;
  --overlay-bg: rgba(255, 255, 255, 0.9);
  --card-bg: #ffffff;
  --card-text: #333;
  --toolbar-bg: rgba(255, 255, 255, 0.95);
  --toolbar-border: #e5e5e5;
  --pill-bg: #fdf2f8;
  --pill-text: #be185d;
  --button-bg: linear-gradient(135deg, #fff2f7, #f4f4f4);
  --button-text: #6b4f5b;
  --button-border: rgba(0, 0, 0, 0.04);
  --button-active-bg: linear-gradient(135deg, #ff7aa2, #ff5d8f);
  --button-active-text: #ffffff;
  --detail-bg: rgba(255, 255, 255, 0.96);
  --detail-border: #f2c7d4;
  --detail-text: #4b4b4b;
  --detail-badge-bg: #fff0f5;
  --detail-badge-text: #be185d;
  --map-bg: #f8f8f8;
  --bubble-bg: linear-gradient(135deg, #fff8fb, #ffe8f0);
  --bubble-border: #ff8fab;
  --bubble-text: #7a3d5b;
  --bubble-title: #ff5d8f;
  --bubble-desc: #8a5a6b;
  --toggle-bg: #ffe4ee;
  --toggle-text: #ac2f61;
}

.page-shell.theme-dark {
  --app-bg: #07111f;
  --text-color: #f8fafc;
  --overlay-bg: rgba(7, 17, 31, 0.88);
  --card-bg: #111c2d;
  --card-text: #f8fafc;
  --toolbar-bg: rgba(10, 20, 36, 0.96);
  --toolbar-border: #283447;
  --pill-bg: #2b2135;
  --pill-text: #ffb0cc;
  --button-bg: linear-gradient(135deg, #26344a, #1d2940);
  --button-text: #f2dfea;
  --button-border: rgba(255,255,255,0.06);
  --button-active-bg: linear-gradient(135deg, #ff7aa2, #ff5d8f);
  --button-active-text: #ffffff;
  --detail-bg: rgba(17, 28, 45, 0.96);
  --detail-border: #4b5a73;
  --detail-text: #e2e8f0;
  --detail-badge-bg: #2b2135;
  --detail-badge-text: #ffb0cc;
  --map-bg: #09111f;
  --bubble-bg: linear-gradient(135deg, #23334a, #18253b);
  --bubble-border: #ff8fab;
  --bubble-text: #f6dfe7;
  --bubble-title: #ff9db5;
  --bubble-desc: #cbd5e1;
  --toggle-bg: #23364d;
  --toggle-text: #f8fafc;
}

.loading-overlay,
.error-overlay {
  position: absolute;
  inset: 0;
  z-index: 20;
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--overlay-bg);
}

.loading-card {
  background: var(--card-bg);
  color: var(--card-text);
  border-radius: 16px;
  padding: 24px 32px;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.12);
  text-align: center;
}

.loading-title {
  margin: 0 0 8px;
  font-size: 1.1rem;
  font-weight: 700;
}

.loading-subtitle {
  margin: 0;
  color: var(--card-text);
  opacity: 0.8;
}

.toolbar {
  padding: 12px 16px;
  background: var(--toolbar-bg);
  border-bottom: 1px solid var(--toolbar-border);
  z-index: 10;
}

.toolbar-inner {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.toolbar-top {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 10px;
}

.toolbar-title {
  margin: 0;
  font-weight: 700;
  color: var(--text-color);
}

.theme-toggle {
  border: none;
  border-radius: 999px;
  padding: 7px 10px;
  background: var(--toggle-bg);
  color: var(--toggle-text);
  font-size: 0.9rem;
  cursor: pointer;
}

.toolbar-meta {
  display: flex;
  align-items: center;
}

.count-pill {
  display: inline-flex;
  align-items: center;
  padding: 6px 10px;
  border-radius: 999px;
  background: var(--pill-bg);
  color: var(--pill-text);
  font-size: 0.9rem;
  font-weight: 700;
}

.category-list {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.category-btn {
  border: 1px solid var(--button-border);
  border-radius: 999px;
  padding: 8px 12px;
  background: var(--button-bg);
  color: var(--button-text);
  font-size: 0.95rem;
  cursor: pointer;
  transition: all 0.2s ease;
  display: inline-flex;
  align-items: center;
  gap: 6px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
}

.category-btn:hover {
  transform: translateY(-1px);
  box-shadow: 0 6px 12px rgba(0, 0, 0, 0.08);
}

.category-btn.active {
  background: var(--button-active-bg);
  color: var(--button-active-text);
  box-shadow: 0 6px 14px rgba(255, 93, 143, 0.25);
}

.emoji {
  font-size: 0.95rem;
}

.detail-panel {
  position: absolute;
  width: min(260px, calc(100% - 32px));
  background: var(--detail-bg);
  color: var(--detail-text);
  border: 1px solid var(--detail-border);
  border-radius: 18px;
  padding: 12px 14px;
  box-shadow: 0 12px 30px rgba(0, 0, 0, 0.14);
  z-index: 30;
  overflow: hidden;
}

.detail-content {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.detail-header {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.detail-badge {
  display: inline-flex;
  align-self: flex-start;
  padding: 4px 8px;
  border-radius: 999px;
  background: var(--detail-badge-bg);
  color: var(--detail-badge-text);
  font-size: 0.78rem;
  font-weight: 700;
}

.detail-description {
  margin: 0;
  color: var(--detail-text);
  font-size: 0.95rem;
  line-height: 1.5;
}

.detail-address {
  margin: 0;
  color: var(--detail-text);
  font-size: 0.9rem;
  line-height: 1.5;
  opacity: 0.9;
}

.detail-image {
  width: 100%;
  max-height: 160px;
  object-fit: cover;
  border-radius: 14px;
  border: 1px solid #f3dce4;
}

.detail-placeholder {
  width: 100%;
  min-height: 120px;
  border-radius: 14px;
  border: 1px dashed #f0c8d6;
  background: linear-gradient(135deg, #fff7fa, #fdf2f8);
  color: #a24a67;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.95rem;
  font-weight: 600;
}

.detail-close {
  position: absolute;
  top: 10px;
  right: 10px;
  border: none;
  background: transparent;
  font-size: 1rem;
  cursor: pointer;
  color: var(--detail-text);
}

.map-area {
  flex: 1;
  min-height: 0;
  background: var(--map-bg);
}

:deep(.cute-bubble) {
  padding: 10px 12px;
  border-radius: 16px;
  background: var(--bubble-bg);
  border: 2px solid var(--bubble-border);
  box-shadow: 0 6px 16px rgba(255, 107, 107, 0.2);
  font-size: 13px;
  color: var(--bubble-text);
  white-space: nowrap;
  transform: translateY(-6px);
}

:deep(.cute-bubble__title) {
  font-weight: 800;
  margin-bottom: 4px;
  color: var(--bubble-title);
}

:deep(.cute-bubble__desc) {
  font-size: 12px;
  color: var(--bubble-desc);
}
</style>
