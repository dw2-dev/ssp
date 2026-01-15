<script setup>
import { ref } from 'vue';

const PAGE_TYPE = {
  GameKey: 'gamekey',
  SignIn: 'signIn',
  SignUp: 'signUp',
  UserView: 'userView',
  ManagerView: 'managerView',
  AdminView: 'adminView',
};

const GAS_URL = "https://script.google.com/macros/s/AKfycbzWm8KNQfsflz2WHzQLWdI2aXypEQLalEuAR6wKC53Qsp10yrtDSNF54qcaUoAzWmeAxg/exec";

const post = async (payload) => {
    try {
        const res = await fetch(GAS_URL, {
            method: "POST",
            body: JSON.stringify(payload),
        });

        const text = await res.text(); // 원시 응답 먼저 읽기
        console.log("🔵 RAW RESPONSE:", text);

        let json;
        try {
            json = JSON.parse(text);
        } catch (e) {
            console.error("❌ JSON 파싱 실패:", e);
            // JSON이 아니면 이렇게 돌려보냄
            return { error: "parse_error", raw: text };
        }

        return json;
    } catch (e) {
        console.error("❌ fetch 에러:", e);
        return { error: "network_error", msg: e.message };
    }
};

const gamekey = ref('');
const showPageType = ref(PAGE_TYPE.GameKey);

const id = ref('');
const pw = ref('');

const adminPw = ref('');

// 회원가입용
const signUpName = ref('');
const signUpPw = ref('');
const gifts = ref([
  { name: '', price: '', link: '' },
  { name: '', price: '', link: '' },
  { name: '', price: '', link: '' },
]);

// 로그인 후 조회용
const myName = ref('');
const myGifts = ref([]);
const mateName = ref('');
const mateGifts = ref([]);

// 관리자용
const adminUsers = ref({});

const onclick_enter = async () => {
  if (!gamekey.value) return alert("다시 말해보시오");

  const res = await post({
    action: "checkGamekey",
    gamekey: gamekey.value
  });

  if (res.error) {
    alert(res.error);
    return;
  }

  showPageType.value = PAGE_TYPE.SignIn;
};

const onclick_return = () => {
  showPageType.value = PAGE_TYPE.SignIn;
  id.value = '';
  pw.value = '';
  adminPw.value = '';
};

const onclick_signUp = () => {
  showPageType.value = PAGE_TYPE.SignUp;
};

const onclick_save = async () => {
  if (!signUpName.value.trim() || !signUpPw.value.trim()) {
    alert("이름과 비밀번호를 입력해 주세요.");
    return;
  }

  const trimmedName = signUpName.value.trim();

  // 입력된 선물만 필터
  const filteredGifts = gifts.value
    .map(g => ({
      name: g.name?.trim() || '',
      price: g.price?.trim() || '',
      link: g.link?.trim() || ''
    }))
    .filter(g => g.name || g.price || g.link);

  if (!filteredGifts.length) {
    const confirmRes = await confirm("원하는 선물이 없는가?");

    if(!confirmRes) return;
  }

  const res = await post({
    action: "register",
    name: trimmedName,
    pw: signUpPw.value,
    gifts: filteredGifts
  });

  if (res.error) {
    switch (res.error) {
      case "invalid_input":
        alert("입력이 올바르지 않습니다.");
        break;
      case "name_too_long":
        alert("이름이 너무 깁니다. 10자 이하로 해주세요.");
        break;
      case "too_much_data":
        alert("선물 정보가 너무 많습니다.");
        break;
      case "exists":
        alert("이미 등록된 이름입니다.");
        break;
      case "limit":
        alert("참가 인원이 꽉 찼습니다.");
        break;
      case "server_error":
        alert("서버 내부 오류가 발생했습니다.\n" + (res.msg || ""));
        break;
      default:
        alert("등록 중 오류가 발생했습니다. 다시 시도해 주세요.");
    }
    return;
  }

  alert("등록 완료! 이제 로그인 해주세요.");

  // 폼 초기화
  signUpName.value = '';
  signUpPw.value = '';
  gifts.value = [
    { name: '', price: '', link: '' },
    { name: '', price: '', link: '' },
    { name: '', price: '', link: '' },
  ];

  onclick_return();
};

const onclick_view = async () => {
  if (!id.value.trim()) return alert("자네, 이름이 뭔가?");
  if (!pw.value.trim()) return alert("비밀번호도 필요하네.");

  // 관리자 진입: id가 'admin'이면 관리자 로그인 플로우
  if (id.value.trim() === 'admin') {
    showPageType.value = PAGE_TYPE.ManagerView;
    pw.value = '';
    return;
  }

  const name = id.value.trim();

  const res = await post({
    action: "login",
    name,
    pw: pw.value
  });

  if (res.error) {
    if (res.error === 'auth') {
      alert("이름 또는 비밀번호가 틀렸습니다.");
    } else {
      alert("로그인 중 오류가 발생했습니다.");
    }
    return;
  }

  myName.value = name;
  myGifts.value = res.me?.gifts || [];
  mateName.value = res.me?.target || '';
  mateGifts.value = res.targetGifts || [];

  id.value = '';
  pw.value = '';

  showPageType.value = PAGE_TYPE.UserView;
};

const onclick_signInManager = async () => {
  if (!adminPw.value.trim()) return alert("여긴 관계자외 출입 금자라네");

  const res = await post({
    action: "adminUsers",
    adminPw: adminPw.value
  });

  if (res.error) {
    alert("관리자 비밀번호가 틀렸습니다.");
    return;
  }

  adminUsers.value = res.users || {};
  showPageType.value = PAGE_TYPE.AdminView;
};

const onclick_loadUsers = async () => {
  if (!adminPw.value.trim()) return alert("관리자 비밀번호를 먼저 입력하세요.");

  const res = await post({
    action: "adminUsers",
    adminPw: adminPw.value
  });

  if (res.error) {
    alert("관리자 권한이 없습니다.");
    return;
  }

  adminUsers.value = res.users || {};
};

const onclick_match = async () => {
  if (!adminPw.value.trim()) return alert("관리자 비밀번호를 입력하세요.");

  if (!confirm("정말 마니또를 배정하시겠습니까?")) return;

  const res = await post({
    action: "match",
    adminPw: adminPw.value
  });

  if (res.error) {
    if (res.error === 'admin_only') {
      alert("관리자 비밀번호가 틀렸습니다.");
    } else if (res.error === 'not_enough_users') {
      alert("참가자가 2명 이상이어야 합니다.");
    } else {
      alert("배정 중 오류가 발생했습니다.");
    }
    return;
  }

  alert("마니또 배정을 완료했습니다.");
  // 최신 정보 다시 로드
  onclick_loadUsers();
};

const onclick_resetAll = async () => {
  if (!adminPw.value.trim()) return alert("관리자 비밀번호를 입력하세요.");

  if (!confirm("정말 전체 리셋하시겠습니까? (정보가 모두 삭제됩니다)")) return;

  const res = await post({
    action: "reset",
    adminPw: adminPw.value
  });

  if (res.error) {
    if (res.error === 'admin_only') {
      alert("관리자 비밀번호가 틀렸습니다.");
    } else {
      alert("리셋 중 오류가 발생했습니다.");
    }
    return;
  }

  alert("전체 리셋을 완료했습니다.");
  adminUsers.value = {};
};
</script>

<template>
    <div class="container">
        <header>
            <h2>🎁 마니또 🎁</h2>
        </header>

        <main>
            <section id="gamekey" v-if="showPageType === PAGE_TYPE.GameKey" class="card fade-in">
                <h3>게임 시작</h3>
                <input v-model="gamekey"
                    @keydown.enter="onclick_enter"
                    placeholder="비밀 암호를 입력하세요" />
                <button class="btn-primary" @click="onclick_enter">입장하기</button>
            </section>

            <section id="signIn" v-if="showPageType === PAGE_TYPE.SignIn" class="card fade-in">
                <h3>로그인</h3>
                <div class="login-grid">
                    <div class="input-group">
                        <input v-model="id" placeholder="이름(ID)" />
                        <input type="password"
                            v-model="pw"
                            placeholder="비밀번호"
                            @keydown.enter="onclick_view"/>
                    </div>
                    <button class="btn-go" @click="onclick_view">GO</button>
                </div>
                <div class="divider">또는</div>
                <button class="btn-secondary" @click="onclick_signUp">새로 참가 등록하기</button>
            </section>

            <section id="signUp" v-if="showPageType === PAGE_TYPE.SignUp" class="card fade-in">
                <h3>참가 등록</h3>
                <div class="input-section">
                    <input v-model="signUpName" placeholder="본인 이름" />
                    <input type="password" v-model="signUpPw" placeholder="사용할 비밀번호" />
                </div>

                <div class="wishlist-header">
                    <h4>위시 리스트 (5만원 이하)</h4>
                    <p>상세히 적을수록 짝꿍이 좋아해요!</p>
                </div>

                <ul class="gift-list">
                    <li v-for="(g, idx) in gifts" :key="idx" class="gift-item">
                        <span class="badge">{{ idx + 1 }}</span>
                        <div class="gift-inputs">
                            <input v-model="g.name" placeholder="선물 이름" />
                            <input v-model="g.price" placeholder="가격(예: 45,000)" />
                            <input v-model="g.link" placeholder="참고 링크(URL)" />
                        </div>
                    </li>
                </ul>

                <div class="button-group">
                    <button class="btn-primary" @click="onclick_save">등록 완료</button>
                    <button class="btn-text" @click="onclick_return">취소하고 돌아가기</button>
                </div>
            </section>

            <section id="userView" v-if="showPageType === PAGE_TYPE.UserView" class="view-container fade-in">
                <div class="status-card">
                    <p class="welcome">안녕하세요, <strong>{{ myName }}</strong>님!</p>
                    <div class="mate-box">
                        <span class="label">내 짝꿍</span>
                        <span class="mate-name" v-if="mateName">{{ mateName }}</span>
                        <span class="mate-name empty" v-else>두근두근... 배정 중!</span>
                    </div>
                </div>

                <div class="wish-grid">
                    <div class="wish-column card">
                        <h4>나의 위시 목록</h4>
                        <div v-if="!myGifts.length" class="empty-msg">등록된 위시가 없습니다.</div>
                        <ul class="result-list">
                            <li v-for="(g, idx) in myGifts" :key="idx">
                                <p class="g-name">{{ g.name }} <span class="g-price">{{ g.price }}</span></p>
                                <a v-if="g.link" :href="g.link" target="_blank" class="link-btn">상품 링크 보기</a>
                            </li>
                        </ul>
                    </div>

                    <div class="wish-column card highlight">
                        <h4>짝꿍의 위시 목록</h4>
                        <div v-if="!mateGifts.length" class="empty-msg">아직 비공개 상태입니다.</div>
                        <ul class="result-list">
                            <li v-for="(g, idx) in mateGifts" :key="idx">
                                <p class="g-name">{{ g.name }} <span class="g-price">{{ g.price }}</span></p>
                                <a v-if="g.link" :href="g.link" target="_blank" class="link-btn">상품 링크 보기</a>
                            </li>
                        </ul>
                    </div>
                </div>

                <button class="btn-secondary" @click="onclick_return">처음으로</button>
            </section>

            <section id="managerView" v-if="showPageType === PAGE_TYPE.ManagerView || showPageType === PAGE_TYPE.AdminView" class="card admin-card fade-in">
                <h3>관리자 모드</h3>

                <div v-if="showPageType === PAGE_TYPE.ManagerView" class="admin-login">
                    <input type="password"
                        v-model="adminPw"
                        placeholder="관리자 암호"
                        @keydown.enter="onclick_signInManager" />
                    <button class="btn-primary" @click="onclick_signInManager">🔓 관리자 인증</button>
                </div>

                <div v-if="showPageType === PAGE_TYPE.AdminView" class="admin-content">
                    <div class="admin-stats">
                        <h4>참가자 현황</h4>
                        <button class="btn-refresh" @click="onclick_loadUsers">🔄 새로고침</button>
                    </div>
                    <ul class="admin-user-list">
                        <li v-for="(u, name) in adminUsers" :key="name">
                            <strong>{{ name }}</strong>
                            <span>🎁 {{ u.gifts?.length || 0 }}</span>
                            <span class="target">🎯 {{ u.target || '미배정' }}</span>
                        </li>
                    </ul>
                    <div class="admin-actions">
                        <button class="btn-primary" @click="onclick_match">🎯 마니또 배정 시작</button>
                        <button class="btn-danger" @click="onclick_resetAll">⚠ 전체 초기화</button>
                    </div>
                </div>
                <button class="btn-text" @click="onclick_return">처음으로</button>
            </section>
        </main>
    </div>
</template>

<style scoped>
/* 기본 배경 및 레이아웃 */
.container {
    min-height: 100dvh;        /* height 대신 min-height */
    box-sizing: border-box;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    padding: 20px;
    background: linear-gradient(135deg, #fdfbfb 0%, #ebedee 100%);
    font-family: 'Pretendard', sans-serif;
    color: #2d3436;
}

header h2 {
    font-size: 2.2rem;
    margin-bottom: 2rem;
    text-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

/* 카드 스타일 */
.card {
    background: white;
    width: 100%;
    max-width: 400px;
    padding: 30px;
    border-radius: 24px;
    box-shadow: 0 15px 35px rgba(0, 0, 0, 0.08);
    box-sizing: border-box;
}

h3 {
    margin-top: 0;
    text-align: center;
    font-size: 1.4rem;
    margin-bottom: 1.5rem;
}

/* 입력창 디자인 */
input {
    width: 100%;
    padding: 14px 18px;
    margin: 8px 0;
    border: 2px solid #f1f3f5;
    border-radius: 12px;
    font-size: 16px;
    transition: all 0.3s ease;
    box-sizing: border-box;
}

input:focus {
    outline: none;
    border-color: #ff7675;
    background-color: #fff;
    box-shadow: 0 4px 12px rgba(255, 118, 117, 0.15);
}

/* 버튼 디자인 */
button {
    width: 100%;
    padding: 15px;
    border-radius: 12px;
    border: none;
    font-size: 16px;
    font-weight: 700;
    cursor: pointer;
    transition: all 0.2s active;
}

.btn-primary {
    background: #ff7675;
    color: white;
    margin-top: 10px;
}

.btn-secondary {
    background: #636e72;
    color: white;
    margin-top: 10px;
}

.btn-go {
    background: #ff7675;
    color: white;
    height: calc(100% - 16px);
    margin-top: 8px;
}

.btn-text {
    background: transparent;
    color: #b2bec3;
    text-decoration: underline;
    margin-top: 15px;
}

.btn-danger {
    background: #ee5253;
    color: white;
    margin-top: 10px;
}

/* 로그인 레이아웃 */
.login-grid {
    display: grid;
    grid-template-columns: 2fr 1fr;
    gap: 10px;
}

.divider {
    display: flex;
    align-items: center;
    color: #b2bec3;
    font-size: 13px;
    margin: 20px 0;
}

.divider::before, .divider::after {
    content: "";
    flex: 1;
    height: 1px;
    background: #eee;
    margin: 0 10px;
}

/* 위시리스트 스타일 */
.wishlist-header {
    margin-top: 25px;
    text-align: center;
}

.wishlist-header h4 { margin-bottom: 5px; }
.wishlist-header p { font-size: 12px; color: #636e72; margin-top: 0; }

.gift-item {
    display: flex;
    gap: 12px;
    margin-bottom: 20px;
    padding-top: 15px;
    border-top: 1px dashed #eee;
}

.badge {
    background: #fab1a0;
    color: white;
    width: 24px;
    height: 24px;
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 12px;
    flex-shrink: 0;
}

.gift-inputs { flex: 1; }
.gift-inputs input { padding: 8px 12px; font-size: 14px; }

/* 사용자 뷰 레이아웃 */
.view-container {
    width: 100%;
    max-width: 800px;
}

.status-card {
    background: white;
    padding: 20px;
    border-radius: 20px;
    text-align: center;
    margin-bottom: 20px;
    box-shadow: 0 8px 20px rgba(0,0,0,0.05);
}

.mate-box {
    margin-top: 10px;
    background: #fff5f5;
    padding: 15px;
    border-radius: 15px;
}

.mate-name {
    display: block;
    font-size: 1.5rem;
    font-weight: 800;
    color: #d63031;
}

.wish-grid {
    display: flex;
    flex-wrap: wrap;
    gap: 20px;
    margin-bottom: 20px;
}

.wish-column {
    flex: 1;
    min-width: 300px;
}

.highlight {
    border: 2px solid #ff7675;
}

.result-list {
    padding: 0;
    list-style: none;
}

.result-list li {
    padding: 12px 0;
    border-bottom: 1px solid #f1f3f5;
}

.g-name { font-weight: 600; margin: 0; }
.g-price { color: #ff7675; font-size: 0.9rem; margin-left: 5px; }

.link-btn {
    display: inline-block;
    margin-top: 5px;
    font-size: 12px;
    color: #0984e3;
    text-decoration: none;
}

/* 관리자 리스트 */
.admin-user-list {
    padding: 0;
    list-style: none;
}

.admin-user-list li {
    display: flex;
    justify-content: space-between;
    padding: 10px;
    background: #f8f9fa;
    margin-bottom: 8px;
    border-radius: 8px;
    font-size: 14px;
}

/* 애니메이션 */
.fade-in {
    animation: fadeIn 0.5s ease-out;
}

@keyframes fadeIn {
    from { opacity: 0; transform: translateY(10px); }
    to { opacity: 1; transform: translateY(0); }
}

/* 모바일 대응 */
@media (max-width: 600px) {
    .wish-column { min-width: 100%; }
    .login-grid { grid-template-columns: 1fr; }
    .btn-go { height: auto; }
}
</style>