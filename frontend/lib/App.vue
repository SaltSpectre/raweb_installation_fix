<!-- Note: Install vue with npm if you want vscode to not show import errors. -->
<!-- First, set your working directiry to the /app folder. Then, run: -->
<!-- npm install vue@3.5.13 vue-router@4.5.0 devalue@5.1.1 -->
<!-- Generated files will be ignored by Git. -->

<script setup lang="ts">
  import { Button, InfoBar, NavigationRail, ProgressRing, TextBlock, Titlebar } from '$components';
  import {
    combineTerminalServersModeEnabled,
    favoritesEnabled,
    registerServiceWorker,
    removeSplashScreen,
    simpleModeEnabled,
    useUpdateDetails,
    useWebfeedData,
  } from '$utils';
  import { hidePortsEnabled } from '$utils/hidePorts';
  import { computed, onMounted, ref, watch, watchEffect } from 'vue';
  import { i18nextPromise } from './i18n';
  import Apps from './pages/Apps.vue';
  import Devices from './pages/Devices.vue';
  import Favorites from './pages/Favorites.vue';
  import Policies from './pages/Policies.vue';
  import Settings from './pages/Settings.vue';
  import Simple from './pages/Simple.vue';

  // TODO: requestClose: remove this logic once all browsers have supported this for some time
  const canUseDialogs = HTMLDialogElement.prototype.requestClose !== undefined;
  const falseWritableComputedRef = computed({
    get: () => false,
    set: () => {},
  });

  // TODO [Anchors]: Remove this when all major browsers support CSS Anchor Positioning
  const supportsAnchorPositions = CSS.supports('position-area', 'center center');

  const webfeedOptions = {
    mergeTerminalServers:
      canUseDialogs === false ? falseWritableComputedRef : combineTerminalServersModeEnabled,
    hidePortsWhenPossible: hidePortsEnabled,
  };
  const { data, loading, error, refresh } = useWebfeedData(window.__iisBase, webfeedOptions);

  // refresh the webfeed when combineTerminalServersModeEnabled changes,
  // but revert the change if there is an error (e.g., if the server is unreachable)
  let combineTerminalServersModeEnabledRevertValue: boolean | null = null;
  watch(combineTerminalServersModeEnabled, async (newValue, oldValue) => {
    // do not refresh if the value has been reverted
    if (newValue === combineTerminalServersModeEnabledRevertValue) {
      return;
    }

    // refresh the webfeed with the new value
    const { error } = await refresh(webfeedOptions);

    // if there is an error, revert the value back to the old value
    // and set the revert value to the old value to prevent an infinite loop
    if (error.value !== null) {
      if (combineTerminalServersModeEnabledRevertValue === null) {
        combineTerminalServersModeEnabledRevertValue = oldValue;
      }
      combineTerminalServersModeEnabled.value = combineTerminalServersModeEnabledRevertValue;
    }

    // if there is no error, set the revert value to null
    // because the value has been successfully changed
    // and we do not need to revert it anymore
    else {
      combineTerminalServersModeEnabledRevertValue = null;
    }
  });

  // refresh the webfeed when hidePortsEnabled changes
  let hidePortsEnabledRevertValue: boolean | null = null;
  watch(hidePortsEnabled, async (newValue, oldValue) => {
    // do not refresh if the value has been reverted
    if (newValue === hidePortsEnabledRevertValue) {
      return;
    }

    // refresh the webfeed with the new value
    const { error } = await refresh(webfeedOptions);

    // if there is an error, revert the value back to the old value
    // and set the revert value to the old value to prevent an infinite loop
    if (error.value !== null) {
      if (hidePortsEnabledRevertValue === null) {
        hidePortsEnabledRevertValue = oldValue;
      }
      hidePortsEnabled.value = hidePortsEnabledRevertValue;
    }

    // if there is no error, set the revert value to null
    // because the value has been successfully changed
    // and we do not need to revert it anymore
    else {
      hidePortsEnabledRevertValue = null;
    }
  });

  const sslError = ref(false);

  const titlebarLoading = ref(false);
  async function listenToServiceWorker(event: any) {
    if (event.data.type === 'fetch-queue') {
      const fetching = event.data.backgroundFetchQueueLength > 0;
      titlebarLoading.value = fetching;
    }
  }

  onMounted(() => {
    registerServiceWorker(listenToServiceWorker).then((response) => {
      if (response === 'SSL_ERROR') {
        sslError.value = true;
      }
    });
  });

  // watch for changes to the URL hash and update the app state accordingly
  const hash = ref(window.location.hash);
  window.addEventListener('hashchange', () => {
    hash.value = window.location.hash;
  });
  watchEffect(() => {
    const allowedRoutes = ['#settings'];

    if (simpleModeEnabled.value) {
      allowedRoutes.unshift('#simple');
    } else {
      allowedRoutes.unshift('#apps', '#devices');
    }

    // add favorites to the allowed routes if enabled
    if (favoritesEnabled.value && !simpleModeEnabled.value && supportsAnchorPositions) {
      allowedRoutes.unshift('#favorites');
    }

    // add policies to the allowed routes if the user is an admin
    if (window.__authUser.isLocalAdministrator) {
      allowedRoutes.push('#policies');
    }

    // if the hash is not recognized, default to the first allowed route
    if (!hash.value || !allowedRoutes.includes(hash.value)) {
      window.location.hash = allowedRoutes[0];
    }
  });

  // track whether i18n is ready
  const i18nReady = ref(false);
  i18nextPromise.then(() => {
    i18nReady.value = true;
  });

  // track whether the component has been mounted
  const mounted = ref(false);
  onMounted(() => {
    mounted.value = true;
  });

  const canRemoveSplashScreen = computed(() => {
    return mounted.value && data.value && !error.value && i18nReady.value;
  });
  watchEffect(() => {
    if (canRemoveSplashScreen.value) {
      removeSplashScreen();
    }
  });

  // track whether the app should show animations, including view transitions
  let prefersReducedMotion = true;
  onMounted(() => {
    const prefersReducedMotionMediaQueryList = window.matchMedia('(prefers-reduced-motion: reduce)');

    function updatePrefersReducedMotion() {
      prefersReducedMotion = prefersReducedMotionMediaQueryList.matches;
    }

    prefersReducedMotionMediaQueryList.addEventListener('change', updatePrefersReducedMotion);
    prefersReducedMotion = prefersReducedMotionMediaQueryList.matches;

    return () => {
      prefersReducedMotionMediaQueryList.removeEventListener('change', updatePrefersReducedMotion);
    };
  });

  // @ts-expect-error window.navigation exists when view transitions are supported
  const navigation = window.navigation;
  if (navigation) {
    // @ts-expect-error navigate event should be typed
    navigation.addEventListener('navigate', (event) => {
      if (
        event.canIntercept &&
        document.startViewTransition &&
        !prefersReducedMotion &&
        canRemoveSplashScreen.value
      ) {
        // if the splash screen is visible, we should not start a view transition
        const splashScreen = document.querySelector<HTMLDivElement>('.root-splash-wrapper');
        const splashScreenVisible = splashScreen && splashScreen.style.display !== 'none';
        if (splashScreenVisible) {
          return;
        }

        const mainElem = document.querySelector('main');
        const mainChildElem = mainElem ? mainElem.querySelector('div') : null;

        // hide overflow so the view transition does not fade between the scroll heights
        if (mainChildElem) {
          mainChildElem.style.overflow = 'hidden';
        }

        const transition = document.startViewTransition();

        // scroll to top between the before transition and the after transition
        transition.ready.then(() => {
          setTimeout(() => {
            if (mainElem && mainChildElem) {
              mainElem.scrollTo({ top: 0, left: 0, behavior: 'instant' });
              mainChildElem.style.overflow = 'unset';
            }
          }, 130);

          requestAnimationFrame(() => {
            // now everything is ready and scroll has happened
            // browser will continue with "after" animations
          });
        });
      }
    });
  }

  const { updateDetails, populateUpdateDetails } = useUpdateDetails();
  onMounted(() => {
    populateUpdateDetails();
  });
</script>

<template>
  <Titlebar forceVisible :loading="titlebarLoading || loading" :update="updateDetails" />
  <div id="appContent">
    <NavigationRail v-if="!simpleModeEnabled" />
    <main :class="{ simple: simpleModeEnabled }">
      <div>
        <template v-if="data">
          <InfoBar
            severity="caution"
            v-if="sslError"
            :title="$t('securityError503.title')"
            style="
              margin: calc(-1 * var(--padding)) calc(-1 * var(--padding)) var(--padding)
                calc(-1 * var(--padding));
              border-radius: 0;
            "
          >
            {{ $t('securityError503.message') }}
            <br />
            <Button
              variant="hyperlink"
              href="https://github.com/kimmknight/raweb/wiki/Trusting-the-RAWeb-server-(Fix-security-error-5003)"
              style="margin-left: -11px; margin-bottom: -6px"
            >
              {{ $t('securityError503.action') }}
            </Button>
          </InfoBar>
          <Favorites :data v-if="hash === '#favorites'" />
          <Devices :data v-else-if="hash === '#devices'" />
          <Apps :data v-else-if="hash === '#apps'" />
          <Simple :data v-else-if="hash === '#simple'" />
          <Settings :update="updateDetails" v-else-if="hash === '#settings'" />
          <Policies :data v-else-if="hash === '#policies'" />
          <div v-else>
            <TextBlock variant="title">404</TextBlock>
            <br />
            <br />
            <TextBlock>Not found</TextBlock>
          </div>
        </template>
        <div v-else>
          <TextBlock variant="title">Loading</TextBlock>
          <br />
          <br />
          <div style="display: flex; gap: 8px; align-items: center">
            <ProgressRing :size="24" />
            <TextBlock style="font-weight: 500">Please wait...</TextBlock>
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<style scoped>
  main {
    flex: 1;
    height: auto;
    overflow: auto;
    background-color: var(--wui-solid-background-tertiary);
    box-sizing: border-box;
    border-radius: var(--wui-overlay-corner-radius) 0 0 0;
  }
  main.simple {
    border-radius: 0;
  }

  main > div {
    --padding: 36px;
    padding: var(--padding);
    width: 100%;
    height: var(--content-height);
    box-sizing: border-box;
    view-transition-name: main;
  }
</style>

<style>
  ::view-transition-group(disabled),
  ::view-transition-old(disabled),
  ::view-transition-new(disabled) {
    animation-duration: 0s !important;
  }

  @keyframes fade-in {
    from {
      opacity: 0;
    }
  }

  @keyframes fade-out {
    to {
      opacity: 0;
    }
  }

  @keyframes entrance {
    from {
      transform: translateY(120px);
      opacity: 0;
    }
  }

  ::view-transition-old(main) {
    animation: var(--wui-view-transition-fade-out) cubic-bezier(0.16, 1, 0.3, 1) both fade-out;
  }

  ::view-transition-new(main) {
    animation: var(--wui-view-transition-fade-in) cubic-bezier(0.16, 1, 0.3, 1)
        var(--wui-view-transition-fade-out) both fade-in,
      var(--wui-view-transition-slide-in) cubic-bezier(0.16, 1, 0.3, 1) both entrance;
  }
</style>
