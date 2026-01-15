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

const GAS_URL = "https://script.google.com/macros/s/AKfycbyt0B7ndeqVr2RSnVSINO1JIvVfddn7S3VAoMGoJRoHqbT3zWyL_OjrYCSNmaTRexTpag/exec";

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

  if (!confirm("정말 전체 리셋하시겠습니까? (선물/매칭 정보가 모두 삭제됩니다)")) return;

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
  <h2>🎁 마니또 🎁</h2>

  <!-- 0. 게임 키 입력 -->
  <div id="gamekey" v-if="showPageType === PAGE_TYPE.GameKey">
    <input
      v-model="gamekey"
      @keydown.enter="onclick_enter"
      placeholder="암호를 대시오"
    />
    <button @click="onclick_enter">입장</button>
  </div>

  <!-- 1. 로그인 -->
  <div id="signIn" class="flex" v-if="showPageType === PAGE_TYPE.SignIn">
    <h3>로그인</h3>
    <div style="display: flex; width: 100%;">
      <div style="display: grid; flex: 1">
        <input
          id="id"
          v-model="id"
          placeholder="이름(id)"
        />
        <input
          id="pw"
          type="password"
          v-model="pw"
          placeholder="비밀번호"
        />
      </div>
      <div style="flex: 1">
        <button
          style="width: 100%; height: 100%; margin-top: 0;"
          @click="onclick_view"
        >
          GO
        </button>
      </div>
    </div>
    <hr style="width: 100%;" />
    <button style="width: 100%;" @click="onclick_signUp">회원가입</button>
  </div>

  <!-- 2. 참가 등록 -->
  <div id="signUp" v-if="showPageType === PAGE_TYPE.SignUp">
    <h3>참가 등록</h3>
    <input
      id="name"
      v-model="signUpName"
      placeholder="이름"
    />
    <input
      id="signUpPw"
      type="password"
      v-model="signUpPw"
      placeholder="비밀번호"
    />

    <hr />
    <small>5만원 이하 선물</small>
    <ul>
      <li v-for="(g, idx) in gifts" :key="idx">
        <input
          :id="`g${idx+1}`"
          v-model="g.name"
          placeholder="이름"
        />
        <input
          :id="`g${idx+1}p`"
          v-model="g.price"
          placeholder="가격"
        />
        <input
          :id="`g${idx+1}l`"
          v-model="g.link"
          placeholder="링크"
        />
        <hr v-if="idx < gifts.length - 1" class="hr-dashed" />
      </li>
    </ul>

    <button @click="onclick_save">등록</button>
    <button @click="onclick_return">처음으로</button>
  </div>

  <!-- 3. 사용자 뷰 -->
  <div id="userView" v-if="showPageType === PAGE_TYPE.UserView">
    <div style="display: flex; gap: 120px;">
      <div id="mine">
        <h3>내 위시 목록 <span v-if="myName">({{ myName }})</span></h3>
        <ul>
          <li v-if="!myGifts.length">등록된 위시가 없습니다.</li>
          <li v-for="(g, idx) in myGifts" :key="idx">
            {{ idx + 1 }}. {{ g.name }} ({{ g.price }}) 
            <br v-if="g.link" />
            <a v-if="g.link" :href="g.link" target="_blank">{{ g.link }}</a>
          </li>
        </ul>
      </div>
      <div id="notMine">
        <h3>내 짝꿍 : <span v-if="mateName">{{ mateName }}</span><span v-else>미배정</span></h3>
        <ul>
          <li v-if="!mateGifts.length">짝꿍 위시는 아직 비밀이거나 없어요.</li>
          <li v-for="(g, idx) in mateGifts" :key="idx">
            {{ idx + 1 }}. {{ g.name }} ({{ g.price }})
            <br v-if="g.link" />
            <a v-if="g.link" :href="g.link" target="_blank">{{ g.link }}</a>
          </li>
        </ul>
      </div>
    </div>
    <div style="display: flex; justify-content: center; margin-top: 20px;">
      <button @click="onclick_return">처음으로</button>
    </div>
  </div>

  <!-- 4. 관리자 진입 (admin id로 로그인했을 때) -->
  <div id="managerView" v-if="showPageType === PAGE_TYPE.ManagerView || showPageType === PAGE_TYPE.AdminView">
    <div v-if="showPageType === PAGE_TYPE.ManagerView">
      <input
        id="adminPw"
        type="password"
        v-model="adminPw"
        placeholder="관리자 비밀번호"
      />
      <button @click="onclick_signInManager">🔓 관리자 확인</button>
      <button @click="onclick_return">처음으로</button>
    </div>

    <!-- 관리자 전용 영역 -->
    <div v-if="showPageType === PAGE_TYPE.AdminView">
      <div style="margin-bottom: 16px;">
        <h3>참가자 목록</h3>
        <button @click="onclick_loadUsers">🔄 새로고침</button>
        <ul v-if="Object.keys(adminUsers).length">
          <li
            v-for="(u, name) in adminUsers"
            :key="name"
          >
            {{ name }} - 위시 {{ u.gifts?.length || 0 }}개 / 타겟:
            {{ u.target || '미배정' }}
          </li>
        </ul>
        <p v-else>참가자 목록이 없습니다. 새로고침을 눌러보세요.</p>
      </div>
      <div>
        <button @click="onclick_match">🎯 마니또 배정</button>
        <button @click="onclick_loadUsers">🤫 기밀 확인</button>
        <button @click="onclick_resetAll">🔄 전체 리셋</button>
      </div>
      <button style="margin-top: 16px;" @click="onclick_return">처음으로</button>
    </div>
  </div>
</template>

<style scoped>
input {
  padding: 12px;
  margin: 5px;
  border-radius: 10px;
  border: 1px solid #eee;
  font-size: 14px;
}
.hr-dashed {
  border: 0px;
  border-top: 2px dashed #444;
}
.flex {
  display: flex;
  flex-direction: column;
  justify-items: center;
  justify-content: center;
  align-items: center;
}
button {
  padding: 12px;
  margin-top: 10px;
  border-radius: 10px;
  border: 1px solid #eee;
  font-size: 14px;
  background: var(--main);
  color: white;
  font-weight: bold;
  cursor: pointer;
}
</style>
