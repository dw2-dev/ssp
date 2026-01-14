<script setup>
  import {ref} from 'vue';

  const PAGE_TYPE = {
    GameKey: 'gamekey',
    SignIn: 'signIn',
    SignUp: 'signUp',
    UserView: 'userView',
    ManagerView: 'managerView',
    AdminView: 'adminView',
  }

  const GAS_URL = "https://script.google.com/macros/s/AKfycbyt0B7ndeqVr2RSnVSINO1JIvVfddn7S3VAoMGoJRoHqbT3zWyL_OjrYCSNmaTRexTpag/exec";
  const post = d => fetch(GAS_URL,{method:"POST",body:JSON.stringify(d)}).then(r=>r.json());

  const gamekey = ref('');
  const showPageType = ref('gamekey');

  const id = ref('');
  const pw = ref('');
  const adminPw = ref('');

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
  }
  const onclick_return = () => {
    showPageType.value = PAGE_TYPE.SignIn;
  }
  const onclick_signUp = () => {
    showPageType.value = PAGE_TYPE.SignUp;
  }
  const onclick_save = () => {
    // save

    onclick_return();
  }
  const onclick_view = () => {
    if (!id.value) return alert("자네, 이름이 뭔가?");
    id.value = '';
    pw.value = '';
    
    showPageType.value = id.value === 'admin' ? PAGE_TYPE.ManagerView : PAGE_TYPE.UserView;
  }
  const onclick_signInManager = () => {
    if (!adminPw.value) return alert("여긴 관계자외 출입 금자라네");

    showPageType.value = PAGE_TYPE.AdminView;
  }
</script>

<template>
  <h2>🎁 마니또 🎁</h2>
  <div id="gamekey"
    v-if="showPageType === PAGE_TYPE.GameKey">
    <input v-model="gamekey"
        @keydown.enter="onclick_enter" 
        placeholder="암호를 대시오" />
    <button @click="onclick_enter">입장</button>
  </div>

  <div id="signIn" class="flex"
    v-if="showPageType === PAGE_TYPE.SignIn">
    <h3>로그인</h3>
    <div style="display: flex; width: 100%;">
        <div style="display: grid; flex:1">
            <input id="id"
                v-model="id"
                placeholder="이름(id)">
            <input id="pw" 
                v-model="pw"
                placeholder="비밀번호">
        </div>
        <div style="flex:1">
            <button style="width: 100%; height: 100%; margin-top: 0;"
                @click="onclick_view">GO</button>
        </div>
    </div>
    <hr style="width: 100%;"/>
    <button style="width: 100%;" @click="onclick_signUp">회원가입</button>
  </div>

  <div id="signUp"
    v-if="showPageType === PAGE_TYPE.SignUp">
    <h3>참가 등록</h3>
    <input id="name" placeholder="이름">
    <input id="pw" placeholder="비밀번호">

    <hr>
    <small>5만원 이하 선물</small>
    <ul>
        <li>
            <input id="g1" placeholder="이름">
            <input id="g1p" placeholder="가격">
            <input id="g1l" placeholder="링크">
        </li>
        <hr class='hr-dashed'/>
        <li>        
            <input id="g2" placeholder="이름">
            <input id="g2p" placeholder="가격">
            <input id="g2l" placeholder="링크">
        </li>
        <hr class='hr-dashed'/>
        <li>
            <input id="g3" placeholder="이름">
            <input id="g3p" placeholder="가격">
            <input id="g3l" placeholder="링크">
        </li>
    </ul>

    <button @click="onclick_save">등록</button>
    <button @click="onclick_return">처음으로</button>
  </div>

  <div id="userView"
    v-if="showPageType === PAGE_TYPE.UserView">
    <div style="display: flex; gap: 120px;">
        <div id="mine">
            <h3>내 위시 목록</h3>
            <ul>
                <li>
                    1번
                </li>
                <li>
                    2번
                </li>
                <li>
                    3번
                </li>
            </ul>
        </div>
        <div id="notMine">
            <h3>내 짝꿍 : </h3>
            <ul>
                <li>
                    1번
                </li>
                <li>
                    2번
                </li>
                <li>
                    3번
                </li>
            </ul>
        </div>
    </div>
    <div style="display: flex; justify-content: center; margin-top: 20px;">
        <button @click="onclick_return">처음으로</button>
    </div>
  </div>
  
  <div id="managerView" v-if="showPageType === PAGE_TYPE.ManagerView || showPageType === PAGE_TYPE.AdminView">
    <div v-if="showPageType === PAGE_TYPE.ManagerView">
        <input id="adminPw"
            v-model="adminPw"
            placeholder="관리자 비밀번호">
        <button @click="onclick_signInManager">🔓 관리자 확인</button>
        <button @click="onclick_return">처음으로</button>
    </div>

    <!-- 관리자 전용 영역 -->
    <div v-if="showPageType === PAGE_TYPE.AdminView">
        <div>
            <ul>
                <li>1</li>
                <li>2</li>
                <li>3</li>
            </ul>
        </div>
        <div>
            <button onclick="match()">🎯 마니또 배정</button>
            <button onclick="match()">기밀 확인</button>
            <button onclick="resetAll()">🔄 전체 리셋</button>
        </div>
        <button @click="onclick_return">처음으로</button>
    </div>

  </div>

</template>

<style scoped>
input {
    padding:12px;
    margin:5px;
    border-radius:10px;
    border:1px solid #eee;
    font-size:14px;
}
.hr-dashed {
    border : 0px;
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
    padding:12px;
    margin-top:10px;
    border-radius:10px;
    border:1px solid #eee;
    font-size:14px;
    background:var(--main);
    color:white;
    font-weight:bold;
    cursor:pointer;
  }
</style>
