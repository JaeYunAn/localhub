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
          <p class="toolbar-title">EODIHOT</p>
          <button class="theme-toggle" @click="toggleTheme">{{ isDarkMode ? '☀️ 라이트' : '🌙 다크' }}</button>
        </div>
        <div v-if="showExplorerControls" class="toolbar-meta">
          <span class="count-pill">선택된 카테고리 수: {{ activeCategories.length }}</span>
          <button class="community-toggle-btn" @click="toggleCommunityPanel">
            {{ showCommunityPanel ? '커뮤니티 닫기' : '커뮤니티 열기' }}
          </button>
        </div>
        <div v-if="showExplorerControls" class="category-list">
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

    <div class="content-area">
      <div id="map" class="map-area"></div>

      <aside v-if="showCommunityPanel" class="community-panel">
        <div class="community-header">
          <div>
            <p class="community-title">{{ selectedPlace?.title || '서울 커뮤니티' }}</p>
            <p class="community-subtitle">서울 권역의 익명 커뮤니티입니다. 제목·내용·비밀번호로 게시글을 저장하고 관리할 수 있습니다.</p>
          </div>
          <button class="community-close" @click="closeCommunityPanel">✕</button>
        </div>

        <div class="community-summary">
          <span>표시 중 {{ visibleCommunityPosts.length }}개</span>
          <span>비밀번호로만 수정/삭제</span>
        </div>

        <div class="community-filter-list">
          <button class="filter-chip" :class="{ active: communityCategoryFilter === '전체' }" @click="setCommunityCategory('전체')">전체</button>
          <button
            v-for="category in categories"
            :key="category"
            class="filter-chip"
            :class="{ active: communityCategoryFilter === category }"
            @click="setCommunityCategory(category)"
          >
            {{ category }}
          </button>
        </div>

        <div class="community-post-list">
          <div v-if="!visibleCommunityPosts.length" class="community-empty">
            아직 작성된 게시글이 없습니다. 첫 게시글을 남겨보세요.
          </div>

          <article
            v-for="post in visibleCommunityPosts"
            :key="post.id"
            class="community-post-card"
            @click="selectCommunityPost(post)"
          >
            <div class="post-meta">
              <span class="post-author">{{ post.author }}</span>
              <span class="post-badge">{{ post.location }}</span>
            </div>
            <h4>{{ post.title }}</h4>
            <p>{{ post.content }}</p>
            <div class="post-footer">
              <span>{{ post.createdAt }}</span>
              <button class="inline-action" type="button" @click.stop="selectCommunityPost(post)">상세</button>
            </div>
          </article>
        </div>

        <div v-if="selectedCommunityPost" class="community-detail">
          <div class="detail-card">
            <div class="detail-head">
              <h4>{{ selectedCommunityPost.title }}</h4>
              <span class="detail-badge">{{ selectedCommunityPost.author }}</span>
            </div>
            <p>{{ selectedCommunityPost.content }}</p>
            <div class="detail-meta">
              <span>{{ selectedCommunityPost.createdAt }}</span>
              <span v-if="selectedCommunityPost.updatedAt">수정 {{ selectedCommunityPost.updatedAt }}</span>
            </div>
            <div class="detail-actions">
              <button type="button" @click="startEditCommunityPost(selectedCommunityPost)">수정</button>
              <button type="button" class="danger" @click="deleteSelectedPost">삭제</button>
            </div>
            <input v-model="communityDeletePassword" type="password" placeholder="삭제 비밀번호" />
          </div>
        </div>

        <form class="community-compose" @submit.prevent="submitCommunityPost">
          <p class="form-title">{{ editingCommunityPostId ? '게시글 수정' : '새 게시글 작성' }}</p>
          <select v-model="communityForm.category">
            <option value="전체">전체</option>
            <option v-for="category in categories" :key="category" :value="category">{{ category }}</option>
          </select>
          <input v-model="communityForm.title" type="text" placeholder="게시글 제목" />
          <textarea v-model="communityForm.content" rows="3" placeholder="서울 여행 경험을 공유해보세요."></textarea>
          <input v-model="communityForm.password" type="password" placeholder="수정/삭제 비밀번호" />
          <div class="form-actions">
            <button type="submit">{{ editingCommunityPostId ? '수정 완료' : '작성 완료' }}</button>
            <button v-if="editingCommunityPostId" type="button" class="secondary" @click="resetCommunityForm">취소</button>
          </div>
        </form>
      </aside>
    </div>

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

    <button class="chat-float-button" @click="toggleChatbot" :class="{ open: chatOpen }">
      <span v-if="!chatOpen">💬</span>
      <span v-else>✕</span>
    </button>

    <div v-if="chatOpen" class="chatbot-panel">
      <div class="chatbot-header">
        <div>
          <p class="chatbot-title">LocalHub AI</p>
          <p class="chatbot-subtitle">서울 여행 정보와 지역 추천을 자연어로 물어보세요.</p>
        </div>
        <button class="chatbot-close" @click="toggleChatbot">✕</button>
      </div>

      <div class="chatbot-messages" ref="chatMessagesRef">
        <div
          v-for="(message, index) in chatMessages"
          :key="index"
          class="chat-message"
          :class="message.role"
        >
          <div class="chat-bubble">{{ message.content }}</div>
        </div>

        <div v-if="chatLoading" class="chat-message assistant">
          <div class="chat-bubble typing">답변을 생성하는 중입니다…</div>
        </div>
      </div>

      <div class="chatbot-footer">
        <p class="chatbot-hint">예: “서울에서 축제 일정 알려줘”, “모범음식점 위치 알려줘”</p>
        <div class="chatbot-input-row">
          <input
            v-model="chatInput"
            type="text"
            placeholder="질문을 입력하세요"
            @keyup.enter.prevent="sendChatMessage"
          />
          <button :disabled="chatLoading" @click="sendChatMessage">전송</button>
        </div>
        <p class="chatbot-warning">API 키는 .env의 VITE_OPENAI_API_KEY로 관리하며, 사용량 제한이 걸린 키만 사용하세요.</p>
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
      seoulLoaded: false,

      chatOpen: false,
      chatInput: '',
      chatLoading: false,
      chatMessages: [
        {
          role: 'assistant',
          content: '안녕하세요! 서울 지역 정보와 축제·추천 질문을 자연어로 물어보세요. 제가 제공된 데이터 기준으로 답변해 드립니다.'
        }
      ],
      showCommunityPanel: false,
      showExplorerControls: false,
      communityCategoryFilter: '전체',
      communityStorageKey: 'localhub-seoul-community-posts',
      communityPosts: [],
      communityForm: {
        title: '',
        content: '',
        password: '',
        category: '관광지'
      },
      editingCommunityPostId: null,
      selectedCommunityPost: null,
      communityDeletePassword: ''
    };
  },
  computed: {
    visibleCommunityPosts() {
      if (this.communityCategoryFilter === '전체') {
        return this.communityPosts;
      }

      return this.communityPosts.filter(post => (post.category || '관광지') === this.communityCategoryFilter);
    }
  },
  watch: {
    activeCategories() {
      this.updatePoiVisibility();
    },
    chatMessages() {
      this.$nextTick(() => this.scrollChatToBottom());
    }
  },
  mounted() {
    this.loadCommunityPosts();
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
      this.isDarkMode = true;
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
      this.showExplorerControls = this.showExplorerControls || title === '서울';
      this.showCommunityPanel = title === '서울';

      this.$nextTick(() => {
        this.positionDetailPanel(position || marker?.getPosition?.());
      });
    },
    closeDetailPanel() {
      this.selectedPlace = null;
      this.showCommunityPanel = false;
      this.detailPanelStyle = { left: '16px', top: '16px' };
    },
    toggleCommunityPanel() {
      this.showCommunityPanel = !this.showCommunityPanel;
    },
    closeCommunityPanel() {
      this.showCommunityPanel = false;
    },
    setCommunityCategory(category) {
      this.communityCategoryFilter = category;
      if (category !== '전체' && !this.editingCommunityPostId) {
        this.communityForm.category = category;
      }
    },
    loadCommunityPosts() {
      if (typeof window === 'undefined') return;
      try {
        const saved = window.localStorage.getItem(this.communityStorageKey);
        if (!saved) {
          this.communityPosts = [];
          return;
        }
        const parsed = JSON.parse(saved);
        this.communityPosts = Array.isArray(parsed) ? parsed : [];
      } catch (error) {
        console.error('커뮤니티 데이터를 불러오지 못했습니다.', error);
        this.communityPosts = [];
      }
    },
    saveCommunityPosts() {
      if (typeof window === 'undefined') return;
      window.localStorage.setItem(this.communityStorageKey, JSON.stringify(this.communityPosts));
    },
    resetCommunityForm() {
      this.communityForm = { title: '', content: '', password: '', category: this.communityCategoryFilter === '전체' ? '관광지' : this.communityCategoryFilter };
      this.editingCommunityPostId = null;
    },
    selectCommunityPost(post) {
      this.selectedCommunityPost = post;
      this.communityDeletePassword = '';
    },
    startEditCommunityPost(post) {
      this.selectedCommunityPost = post;
      this.editingCommunityPostId = post.id;
      this.communityForm = {
        title: post.title,
        content: post.content,
        password: '',
        category: post.category || '관광지'
      };
    },
    submitCommunityPost() {
      const title = this.communityForm.title.trim();
      const content = this.communityForm.content.trim();
      const password = this.communityForm.password.trim();

      if (!title || !content || !password) {
        window.alert('제목, 내용, 비밀번호를 모두 입력해주세요.');
        return;
      }

      if (this.editingCommunityPostId) {
        const target = this.communityPosts.find(post => post.id === this.editingCommunityPostId);
        if (!target) {
          this.resetCommunityForm();
          return;
        }

        if (target.password !== password) {
          window.alert('비밀번호가 일치하지 않아 수정할 수 없습니다.');
          return;
        }

        target.title = title;
        target.content = content;
        target.password = password;
        target.category = this.communityForm.category || target.category || '관광지';
        target.updatedAt = '수정됨';
        this.selectedCommunityPost = { ...target };
      } else {
        const createdAt = new Date().toLocaleString('ko-KR', {
          year: 'numeric',
          month: '2-digit',
          day: '2-digit',
          hour: '2-digit',
          minute: '2-digit'
        });

        this.communityPosts.unshift({
          id: Date.now(),
          location: '서울',
          author: '익명',
          title,
          content,
          password,
          category: this.communityForm.category || '관광지',
          createdAt,
          updatedAt: null
        });
        this.selectedCommunityPost = this.communityPosts[0];
      }

      this.saveCommunityPosts();
      this.resetCommunityForm();
    },
    deleteSelectedPost() {
      if (!this.selectedCommunityPost) return;
      const password = this.communityDeletePassword.trim();
      if (!password) {
        window.alert('삭제를 위해 비밀번호를 입력해주세요.');
        return;
      }

      if (this.selectedCommunityPost.password !== password) {
        window.alert('비밀번호가 일치하지 않아 삭제할 수 없습니다.');
        return;
      }

      this.communityPosts = this.communityPosts.filter(post => post.id !== this.selectedCommunityPost.id);
      this.saveCommunityPosts();
      this.selectedCommunityPost = null;
      this.communityDeletePassword = '';
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
        level: 12
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
          this.closeDetailPanel();
          if (r.name === '서울') {
            this.showExplorerControls = true;
            this.showCommunityPanel = true;
            this.loadSeoulPois();
          } else {
            this.showExplorerControls = false;
            this.showCommunityPanel = false;
          }
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
    },

    toggleChatbot() {
      this.chatOpen = !this.chatOpen;
      if (this.chatOpen) {
        this.$nextTick(() => this.scrollChatToBottom());
      }
    },
    scrollChatToBottom() {
      const container = this.$refs.chatMessagesRef;
      if (container) {
        container.scrollTop = container.scrollHeight;
      }
    },
    async sendChatMessage() {
      const message = this.chatInput.trim();
      if (!message || this.chatLoading) return;

      this.chatMessages.push({ role: 'user', content: message });
      this.chatInput = '';
      this.chatLoading = true;

      try {
        const answer = await this.getChatbotAnswer(message);
        this.chatMessages.push({ role: 'assistant', content: answer });
      } catch (error) {
        this.chatMessages.push({ role: 'assistant', content: '답변 생성 중 오류가 발생했습니다. 잠시 후 다시 시도해 주세요.' });
      } finally {
        this.chatLoading = false;
      }
    },
    buildKnowledgeContext() {
      return Object.entries(this.seoulDataByCategory)
        .map(([category, items]) => {
          const sample = (items || []).slice(0, 8).map(item => {
            const title = item.title || '제목 없음';
            const address = item.addr1 || item.addr2 || '주소 미제공';
            const overview = item.overview || item.description || '';
            return `- ${title} | ${address} | ${overview}`.trim();
          }).join('\n');

          return `${category}\n${sample}`;
        })
        .join('\n\n');
    },
    buildChatMessages(query) {
      const context = this.buildKnowledgeContext();
      return [
        {
          role: 'system',
          content: '당신은 서울 지역 여행 정보 도우미입니다. 아래에 제공된 데이터만 근거로 답하세요. 사실이 없으면 모른다고 말하고, 질문에 맞는 카테고리와 위치를 우선적으로 제시하세요.'
        },
        {
          role: 'user',
          content: `질문: ${query}\n\n데이터:\n${context}`
        }
      ];
    },
    async getChatbotAnswer(query) {
      const apiKey = import.meta.env.VITE_OPENAI_API_KEY;
      if (!apiKey || apiKey === 'your_openai_api_key_here' || apiKey === 'YOUR_OPENAI_API_KEY') {
        return this.getLocalFallbackAnswer(query);
      }

      const response = await fetch('https://api.openai.com/v1/chat/completions', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${apiKey}`
        },
        body: JSON.stringify({
          model: 'gpt-4o-mini',
          temperature: 0.2,
          messages: this.buildChatMessages(query)
        })
      });

      if (!response.ok) {
        throw new Error('OpenAI 응답 실패');
      }

      const data = await response.json();
      return data?.choices?.[0]?.message?.content?.trim() || '답변을 만들지 못했습니다.';
    },
    getLocalFallbackAnswer(query) {
      const q = query.toLowerCase();

      if (q.includes('축제') || q.includes('행사')) {
        return this.formatResults('축제공연행사', q, '축제/행사');
      }

      if (q.includes('맛집') || q.includes('음식') || q.includes('식당') || q.includes('모범')) {
        return this.formatResults('쇼핑', q, '맛집 또는 음식점');
      }

      if (q.includes('여행') || q.includes('추천') || q.includes('관광')) {
        return this.formatResults('관광지', q, '관광지');
      }

      const categories = Object.keys(this.seoulDataByCategory).join(', ');
      return `현재는 로컬 데이터 기반으로 바로 답할 수 있습니다. 예시로 ${categories} 정보를 조회할 수 있어요. 더 구체적인 질문을 보내 주세요.`;
    },
    formatResults(category, query, label) {
      const items = (this.seoulDataByCategory[category] || []).filter(item => {
        const text = `${item.title || ''} ${item.addr1 || ''} ${item.overview || ''}`.toLowerCase();
        return text.includes(query) || query.split(/\s+/).every(keyword => text.includes(keyword));
      });

      if (!items.length) {
        return `${label} 관련 데이터를 찾지 못했습니다. 다른 키워드로 다시 질문해 주세요.`;
      }

      const topItems = items.slice(0, 3);
      const lines = topItems.map(item => `- ${item.title || '이름 없음'} (${item.addr1 || '주소 미제공'})`).join('\n');
      return `${label} 관련 추천 결과입니다.\n${lines}`;
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
  font-size: clamp(2rem, 4.5vw, 3rem);
  font-weight: 900;
  letter-spacing: 0.14em;
  color: var(--text-color);
  text-transform: uppercase;
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
  justify-content: space-between;
  gap: 10px;
}

.community-toggle-btn {
  border: none;
  border-radius: 999px;
  padding: 8px 12px;
  background: linear-gradient(135deg, #ff7aa2, #ff5d8f);
  color: white;
  font-weight: 700;
  cursor: pointer;
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

.content-area {
  flex: 1;
  display: flex;
  min-height: 0;
  position: relative;
}

.map-area {
  flex: 1;
  min-width: 0;
  height: 100%;
}

.community-panel {
  width: min(360px, 100%);
  display: flex;
  flex-direction: column;
  background: rgba(255, 255, 255, 0.97);
  border-left: 1px solid #e5e5e5;
  box-shadow: -8px 0 24px rgba(0, 0, 0, 0.06);
  z-index: 20;
}

.community-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 12px;
  padding: 16px;
  border-bottom: 1px solid #f1f1f1;
}

.community-title {
  margin: 0 0 6px;
  font-size: 1rem;
  font-weight: 800;
  color: #222;
}

.community-subtitle {
  margin: 0;
  font-size: 0.9rem;
  color: #6b7280;
  line-height: 1.5;
}

.community-close {
  border: none;
  background: transparent;
  cursor: pointer;
  font-size: 1rem;
  color: #6b7280;
}

.community-summary {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0 16px 12px;
  font-size: 0.8rem;
  color: #6b7280;
}

.community-filter-list {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  padding: 0 16px 12px;
}

.filter-chip {
  border: 1px solid #f2c7d4;
  border-radius: 999px;
  padding: 6px 10px;
  background: #fff;
  color: #6b7280;
  font-size: 0.8rem;
  cursor: pointer;
}

.filter-chip.active {
  background: linear-gradient(135deg, #ff7aa2, #ff5d8f);
  color: white;
  border-color: #ff7aa2;
}

.community-post-list {
  flex: 1;
  overflow-y: auto;
  padding: 0 16px 12px;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.community-empty {
  border: 1px dashed #f0c7d6;
  border-radius: 12px;
  padding: 16px;
  text-align: center;
  color: #8b5e72;
  background: #fff7fa;
}

.community-post-card {
  border: 1px solid #f0dbe4;
  border-radius: 14px;
  padding: 12px;
  background: #fff;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.04);
  cursor: pointer;
}

.post-meta,
.post-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 8px;
  font-size: 0.8rem;
  color: #6b7280;
}

.post-author {
  font-weight: 700;
  color: #be185d;
}

.post-badge {
  padding: 4px 8px;
  border-radius: 999px;
  background: #fdf2f8;
  color: #be185d;
  font-size: 0.75rem;
}

.community-post-card h4 {
  margin: 8px 0 6px;
  font-size: 0.95rem;
  color: #111827;
}

.community-post-card p {
  margin: 0 0 8px;
  font-size: 0.88rem;
  color: #4b5563;
  line-height: 1.5;
}

.inline-action {
  border: none;
  background: transparent;
  color: #be185d;
  font-weight: 700;
  cursor: pointer;
}

.community-detail {
  padding: 0 16px 12px;
}

.detail-card {
  border: 1px solid #f3d8e2;
  border-radius: 14px;
  padding: 12px;
  background: linear-gradient(135deg, #fff8fb, #ffffff);
}

.detail-head {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 8px;
}

.detail-card h4 {
  margin: 0;
  font-size: 0.95rem;
  color: #111827;
}

.detail-badge {
  font-size: 0.75rem;
  padding: 4px 8px;
  border-radius: 999px;
  background: #fdf2f8;
  color: #be185d;
}

.detail-card p {
  margin: 8px 0;
  font-size: 0.88rem;
  color: #4b5563;
  line-height: 1.5;
}

.detail-meta {
  display: flex;
  justify-content: space-between;
  gap: 8px;
  font-size: 0.75rem;
  color: #6b7280;
  margin-bottom: 8px;
}

.detail-actions {
  display: flex;
  gap: 8px;
  margin-bottom: 8px;
}

.detail-actions button,
.form-actions button {
  border: none;
  border-radius: 10px;
  padding: 8px 10px;
  cursor: pointer;
  font-weight: 700;
}

.detail-actions button {
  background: #fdf2f8;
  color: #be185d;
}

.detail-actions .danger,
.form-actions .secondary {
  background: #fef2f2;
  color: #b91c1c;
}

.community-compose {
  display: flex;
  flex-direction: column;
  gap: 8px;
  padding: 12px 16px 16px;
  border-top: 1px solid #f1f1f1;
}

.form-title {
  margin: 0;
  font-weight: 700;
  color: #111827;
}

.community-compose select,
.community-compose input,
.community-compose textarea {
  border: 1px solid #e5e7eb;
  border-radius: 10px;
  padding: 10px 12px;
  font: inherit;
  resize: vertical;
}

.form-actions {
  display: flex;
  gap: 8px;
}

.form-actions button[type='submit'] {
  background: linear-gradient(135deg, #ff7aa2, #ff5d8f);
  color: white;
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

.chat-float-button {
  position: absolute;
  right: 16px;
  bottom: 16px;
  width: 56px;
  height: 56px;
  border: none;
  border-radius: 50%;
  background: linear-gradient(135deg, #ff7aa2, #ff5d8f);
  color: white;
  font-size: 1.3rem;
  cursor: pointer;
  box-shadow: 0 8px 20px rgba(255, 93, 143, 0.25);
  z-index: 40;
}

.chat-float-button.open {
  background: linear-gradient(135deg, #52525b, #3f3f46);
}

.chatbot-panel {
  position: absolute;
  right: 16px;
  bottom: 84px;
  width: min(360px, calc(100% - 32px));
  height: min(520px, calc(100vh - 140px));
  display: flex;
  flex-direction: column;
  background: rgba(255, 255, 255, 0.98);
  border: 1px solid #f1d7e2;
  border-radius: 20px;
  box-shadow: 0 16px 40px rgba(0, 0, 0, 0.16);
  overflow: hidden;
  z-index: 45;
}

.chatbot-header {
  padding: 14px 16px;
  border-bottom: 1px solid #f5e1ea;
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: linear-gradient(135deg, #fff8fb, #ffe8f0);
}

.chatbot-title {
  margin: 0;
  font-size: 1rem;
  font-weight: 800;
  color: #8a2146;
}

.chatbot-subtitle {
  margin: 4px 0 0;
  font-size: 0.8rem;
  color: #7a5160;
}

.chatbot-close {
  border: none;
  background: transparent;
  font-size: 1rem;
  cursor: pointer;
}

.chatbot-messages {
  flex: 1;
  padding: 12px;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 8px;
  background: #fffdfd;
}

.chat-message {
  display: flex;
}

.chat-message.user {
  justify-content: flex-end;
}

.chat-bubble {
  max-width: 90%;
  padding: 10px 12px;
  border-radius: 14px;
  font-size: 0.95rem;
  line-height: 1.5;
  white-space: pre-wrap;
}

.chat-message.assistant .chat-bubble {
  background: #fff0f5;
  color: #7a3d5b;
}

.chat-message.user .chat-bubble {
  background: linear-gradient(135deg, #ff7aa2, #ff5d8f);
  color: white;
}

.chatbot-footer {
  padding: 12px;
  border-top: 1px solid #f5e1ea;
  background: white;
}

.chatbot-hint {
  margin: 0 0 8px;
  font-size: 0.8rem;
  color: #8a6a75;
}

.chatbot-input-row {
  display: flex;
  gap: 8px;
}

.chatbot-input-row input {
  flex: 1;
  border: 1px solid #f0d9e2;
  border-radius: 999px;
  padding: 10px 12px;
  outline: none;
}

.chatbot-input-row button {
  border: none;
  border-radius: 999px;
  padding: 10px 14px;
  background: linear-gradient(135deg, #ff7aa2, #ff5d8f);
  color: white;
  cursor: pointer;
}

.chatbot-input-row button:disabled {
  opacity: 0.7;
  cursor: wait;
}

.chatbot-warning {
  margin: 8px 0 0;
  font-size: 0.74rem;
  color: #b47b8d;
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