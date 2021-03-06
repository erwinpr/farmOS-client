<template>
  <div>
    <transition name="filter">
      <div class="modal-filter" v-if="showDrawer" @click="showDrawer = !showDrawer"/>
    </transition>

    <transition name="drawer">
      <div class="drawer container-fluid" v-if="showDrawer">
        <header class="drawer-header row">
          <div class="col">
            <div class="arrow-back" @click="showDrawer = !showDrawer">
              <icon-arrow-back/>
            </div>
            <div v-if="isLoggedIn" class="user-info">
              <h2>{{ farmName }}</h2>
              <p>{{ farmUrl }}</p>
              <p>{{ username}}</p>
            </div>
            <div v-else class="user-info">
              <h2>{{ $t('Welcome to farmOS') }}</h2>
              <p>{{ $t('Login for more options.') }}</p>
            </div>
          </div>
        </header>
        <farm-list>
          <router-link to="/home" @click.native="showDrawer = !showDrawer">
            <farm-list-item>{{ $t('Home') }}</farm-list-item>
          </router-link>
        </farm-list>
        <farm-list>
          <farm-list-item
            v-for="module in modules"
            :key="`${module.name}-menu-link`"
            @click="handleModuleClick(module)">
            {{$t(module.label)}}
          </farm-list-item>
        </farm-list>
        <farm-list>
          <farm-list-item :clickable="false">
            <farm-toggle-check
              :label="$t('Share My Location')"
              labelPosition="before"
              :checked="useGeolocation"
              @input="setUseGeolocation($event)"/>
          </farm-list-item>
          <farm-list-item v-if="langs.length > 0">
            <label>{{$t('Select Language')}}</label><br>
            <div class="form-check">
              <input
                type="radio"
                class="form-check-input"
                name="language"
                id="lang1"
                value="en"
                @input="setLocale"
                :checked="isLocale('en')"/>
              <label for="lang1"  class="form-check-label">
                {{ $t('English') }}
              </label>
            </div>
            <div
              v-for="(lang, i) in langs"
              :key="`lang-${i}`"
              class="form-check">
              <input
                type="radio"
                class="form-check-input"
                name="language"
                :id="`lang-${i}`"
                :value="lang.language"
                @input="setLocale"
                :checked="isLocale(lang.language)"/>
              <label :for="`lang-${i}`"  class="form-check-label">
                {{ lang.native }}
              </label>
            </div>
          </farm-list-item>
          <farm-list-item :clickable="false">{{ $t('Version')}}: {{version}}</farm-list-item>
          <router-link to="/login" v-if="!isLoggedIn" @click.native="showDrawer = !showDrawer">
            <farm-list-item >{{ $t('Login') }}</farm-list-item>
          </router-link>
          <router-link to="/logout" v-if="isLoggedIn" @click.native="showDrawer = !showDrawer">
            <farm-list-item>{{ $t('Logout') }}</farm-list-item>
          </router-link>
        </farm-list>

      </div>
    </transition>

    <div class="module-container">
      <router-view
        name="menubar"
        @toggle-drawer="showDrawer = !showDrawer"
      />
      <router-view
        :useGeolocation="useGeolocation"
        :userId='userId'
        :systemOfMeasurement='systemOfMeasurement'
        :logTypes='logTypes'
        :modules="modules"
        :logs='logs'
        :areas='areas'
        :assets='assets'
        :units='units'
        :categories='categories'
        :equipment='equipment'
        :areaGeoJSON='areaGeoJSON'
        @toggle-drawer="showDrawer = !showDrawer"
      />
    </div>

    <div
      v-for="(err, index) in errors"
      :key="`alert-${index}`"
      class="alerts">
      <div
        class="alert alert-warning alert-dismissable" >
        <span v-html="err.message"></span>
        <button
          type="button"
          @click="closeAlert(index)"
          class="close"
          aria-label="Close">
          <span aria-hidden="true">&times;</span>
        </button>
      </div>
    </div>

  </div>
</template>

<script>
import { mapState, mapGetters } from 'vuex';
import { version } from '../../package.json';

export default {
  name: 'App',
  data() {
    return {
      showDrawer: false,
      version,
    };
  },
  created() {
    this.$store.commit('setCurrentModule', this.$route.meta.module);
    this.$store.dispatch('loadCachedUserAndSiteInfo');
    this.$store.dispatch('loadCachedResources');
    this.$store.dispatch('loadCachedLanguages');
    this.$store.dispatch('updateUserAndSiteInfo')
      .then(res => this.$store.dispatch('updateFarmResources', res))
      .then(res => this.$store.dispatch('updateLanguages', res));
    this.$store.dispatch('loadCachedAssets');
    this.$store.dispatch('loadCachedAreas');
    this.$store.dispatch('loadCachedUnits');
    this.$store.dispatch('loadCachedCategories');
    this.$store.dispatch('purgeLogs');
  },
  computed: {
    ...mapState({
      /**
       * SHELL STATE
       */
      errors: state => state.shell.errors,
      username: state => state.shell.user.name,
      userId: state => state.shell.user.uid,
      isLoggedIn: state => state.shell.user.isLoggedIn,
      useGeolocation: state => state.shell.settings.useGeolocation,
      systemOfMeasurement: state => state.shell.systemOfMeasurement,
      farmName: state => state.shell.farmInfo.name,
      // Provide an example url for the dev server environment
      farmUrl: state => ((state.shell.farmInfo.url === '')
        ? 'example.farmos.net'
        : state.shell.farmInfo.url?.replace(/(^\w+:|^)\/\//, '')),
      modules: state => state.shell.modules,
      areaGeoJSON: state => state.shell.areaGeoJSON,

      /**
       * L10N STATE
       */
      locale: state => state.l10n.locale,
      langs: state => state.l10n.languages,

      /**
       * FARM STATE
       */
      logs: state => state.farm.logs,
      areas: state => state.farm.areas,
      assets: state => state.farm.assets,
      units: state => state.farm.units,
      categories: state => state.farm.categories,
      logTypes: state => state.farm.resources.log,
    }),
    ...mapGetters([
      'equipment',
    ]),
  },
  watch: {
    showDrawer(currentShowDrawer) {
      if (currentShowDrawer) {
        document.querySelector('body').setAttribute('style', 'overflow-y: hidden');
      } else {
        document.querySelector('body').setAttribute('style', 'overflow-y: visible');
      }
    },
    $route(to) {
      this.$store.commit('setCurrentModule', to.meta.module);
    },
  },
  methods: {
    closeAlert(index) {
      this.$store.commit('dismissAlert', index);
    },
    setUseGeolocation(checked) {
      this.$store.commit('setUseGeolocation', checked);
    },
    handleModuleClick(module) {
      if (module.routes[0].path !== this.$route.path) {
        this.$router.push(module.routes[0].path);
      }
      this.showDrawer = !this.showDrawer;
    },
    setLocale(e) {
      this.$store.commit('setLocale', e.target.value);
    },
    isLocale(locale) {
      return this.locale === locale;
    },
  },
};
</script>

<style scoped>
  .close {
    position: absolute;
    top: 5px;
    right: 5px;
  }

  .drawer {
    position: fixed;
    top: 0;
    height: 100vh;
    width: 80vw;
    background-color: var(--white);
    z-index: 2000;
    overflow-y: auto;
  }

  .modal-filter {
    position: fixed;
    top: 0;
    height: 100%;
    width: 100%;
    background-color: var(--dark-transparent);
    z-index: 1500;
  }

  .drawer-enter, .drawer-leave-to {
    transform: translateX(-80vw);
  }

  .filter-enter, .filter-leave-to {
    opacity: 0;
  }

  .drawer-enter-active, .drawer-leave-active,
  .filter-enter-active, .filter-leave-active {
    transition: all .3s ease;
  }

  .drawer header {
    background-color: var(--primary);
    color: var(--white);
    height: 9rem;
  }

  .arrow-back svg {
    fill: var(--white);
  }

  .arrow-back {
    margin-top: 0.5rem;
  }

  .drawer a, .drawer a:hover {
    text-decoration: none;
    color: inherit;
  }

  .alerts {
    position: absolute;
    z-index: 1001;
    bottom: var(--s);
    right: var(--s);
    left: var(--s);
    text-align: center;
  }

  .alert {
    display: inline-block;
    box-shadow: var(--shadow-strong);
  }

</style>
