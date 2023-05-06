<template>
  <main @click="startTimer = !startTimer">
    <div class="auth" v-if="!token">
      Авторизация...
    </div>
    <div class="timer" v-else>
      {{timer}}
    </div>
  </main>
</template>

<script>
import { computed } from "@vue/reactivity";
import { defineComponent, ref } from "vue"


export default defineComponent({
  name: 'App',
  components: {},

  setup() {
    const appId = 10977;
    const redirectUrl = 'https://redpondaa.github.io/Donaton/'
    const hash = new URLSearchParams((new URL(window.location.toString())).hash.slice(1));

    let rublePerMin = 10;
    let currentTimer = ref(0);
    let lastAlertId = 0;
    let startTimer = ref(false);

    if (hash.has('rublePerMin')) {
      rublePerMin = parseInt(hash.get('rublePerMin'));
      localStorage.setItem('rublePerMin', rublePerMin);
    }
    else if (localStorage['rublePerMin']) {
      rublePerMin = parseInt(localStorage['rublePerMin']);
    }

    if (hash.has('timer')) {
      currentTimer.value = parseInt(hash.get('timer'));
      localStorage.setItem('timer', currentTimer.value);
    }
    else if (localStorage['timer']) {
      currentTimer.value = parseInt(localStorage['timer']);
    }

    if (hash.has('lastAlertId')) {
      lastAlertId = parseInt(hash.get('lastAlertId'));
      localStorage.setItem('lastAlertId', lastAlertId);
    }
    else if (localStorage['lastAlertId']) {
      lastAlertId = parseInt(localStorage['lastAlertId']);
    }

    let token = hash.get('access_token');

    if (!token) {
      window.location = `https://www.donationalerts.com/oauth/authorize?client_id=${appId}&redirect_uri=${encodeURI(redirectUrl)}&response_type=token&scope=oauth-donation-subscribe oauth-donation-index oauth-user-show`
    }

    const checkLastDonates = async () => {
      const response = await (await fetch(
        `https://www.donationalerts.com/api/v1/alerts/donations?access_token=${token}`,
        {
          headers: {
            'Authorization': `Bearer ${token}`
          }
        }
        )).json();

      if (response.data.length > 0) {
        const lastDonate = response.data[0];
        if (lastDonate.id > lastAlertId) {
          response.data.forEach(donate => {
            if (donate.id > lastAlertId) {
              lastAlertId = donate.id;
              localStorage.setItem('lastAlertId', lastAlertId);

              currentTimer.value += donate.amount_in_user_currency / rublePerMin * 60;
              localStorage.setItem('timer', currentTimer.value);
            }
          });

          lastAlertId = lastDonate.id;
          localStorage.setItem('lastAlertId', lastAlertId);
        }
      }
    }

    const timer = computed(() => {
      const days = (currentTimer.value / (24 * 60 * 60));
      return `${Math.floor(days)}d ${new Date(currentTimer.value * 1000).toISOString().substr(11, 8)}`;
    });

    checkLastDonates();
    setInterval(() => {
      checkLastDonates();
    }, 5000);

    setInterval(() => {
      if (startTimer.value) {
        if (currentTimer.value > 0) {
          currentTimer.value--;
          localStorage.setItem('timer', currentTimer.value);
        }
      }
    }, 1000);

    return {token, currentTimer, timer, startTimer}
  }
})
</script>

<style lang="scss">
html, body {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #333;
  display: flex;
  justify-content: center;
  align-items: center;
  left: 0;
  top: 0;
  padding: 0;
  margin: 0;
  width: 100vw;
  height: 100vh;
}
</style>
