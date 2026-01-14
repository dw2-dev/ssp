<script setup>
  import MainView from './components/MainView.vue';
</script>

<template>
  <div class="main">
    <MainView />
  </div>
</template>

<style scoped>
  .main {
    min-height: 90vh;  
    min-width: 50vw;
  }
  div {
    display: flex;
    flex-direction: column;
    justify-items: center;
    justify-content: center;
    align-items: center;
  }
  button {
    width:100%;
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
