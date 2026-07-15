<template>
  <div class="page-shell">
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
        <p class="toolbar-title">카카오맵이 먼저 보이고, 기능은 그 다음에 표시됩니다.</p>
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

    <div v-if="selectedPlace" class="detail-panel">
      <button class="detail-close" @click="selectedPlace = null">✕</button>
      <h3>{{ selectedPlace.title }}</h3>
      <p>{{ selectedPlace.description }}</p>
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
      ]
    };
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
    toggleCategory(category) {
      if (this.activeCategories.includes(category)) {
        this.activeCategories = this.activeCategories.filter(item => item !== category);
      } else {
        this.activeCategories = [...this.activeCategories, category];
      }
    },
    showBubble(marker, title, description) {
      if (this.customOverlay) {
        this.customOverlay.setMap(null);
      }
      this.customOverlay = null;
      this.selectedPlace = { title, description };
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
          this.showBubble(marker, r.name, r.name === '서울' ? '서울 여행지 데이터를 불러왔어요.' : `${r.name} 지역의 여행 정보를 확인해보세요.`);
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
            this.showBubble(marker, item.title, item.addr1 || '서울 여행지예요.');
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
}

.loading-overlay,
.error-overlay {
  position: absolute;
  inset: 0;
  z-index: 20;
  display: flex;
  align-items: center;
  justify-content: center;
  background: rgba(255, 255, 255, 0.9);
}

.loading-card {
  background: white;
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
  color: #666;
}

.toolbar {
  padding: 12px 16px;
  background: rgba(255, 255, 255, 0.95);
  border-bottom: 1px solid #e5e5e5;
  z-index: 10;
}

.toolbar-inner {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.toolbar-title {
  margin: 0;
  font-weight: 700;
  color: #222;
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
  background: #fdf2f8;
  color: #be185d;
  font-size: 0.9rem;
  font-weight: 700;
}

.category-list {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.category-btn {
  border: none;
  border-radius: 999px;
  padding: 8px 12px;
  background: linear-gradient(135deg, #fff2f7, #f4f4f4);
  color: #6b4f5b;
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
  background: linear-gradient(135deg, #ff7aa2, #ff5d8f);
  color: white;
  box-shadow: 0 6px 14px rgba(255, 93, 143, 0.25);
}

.emoji {
  font-size: 0.95rem;
}

.detail-panel {
  position: absolute;
  right: 16px;
  bottom: 16px;
  width: min(320px, calc(100% - 32px));
  background: rgba(255, 255, 255, 0.96);
  border: 1px solid #f2c7d4;
  border-radius: 16px;
  padding: 16px;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.12);
  z-index: 30;
}

.detail-close {
  position: absolute;
  top: 8px;
  right: 8px;
  border: none;
  background: transparent;
  font-size: 1rem;
  cursor: pointer;
}

.map-area {
  flex: 1;
  min-height: 0;
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
  background: linear-gradient(135deg, #fff8fb, #ffe8f0);
  border: 2px solid #ff8fab;
  box-shadow: 0 6px 16px rgba(255, 107, 107, 0.2);
  font-size: 13px;
  color: #7a3d5b;
  white-space: nowrap;
  transform: translateY(-6px);
}

:deep(.cute-bubble__title) {
  font-weight: 800;
  margin-bottom: 4px;
  color: #ff5d8f;
}

:deep(.cute-bubble__desc) {
  font-size: 12px;
  color: #8a5a6b;
}
</style>