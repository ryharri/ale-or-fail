<script>
  import { App } from 'framework7-svelte';
  import { onMount } from 'svelte';
  import HomePage from './pages/HomePage.svelte';
  import AleOrFail from './pages/AleOrFail.svelte';

  const normalizeRoute = (value = window.location.hash || '#/') => {
    const hash = value.startsWith('#') ? value.slice(1) : value;
    if (!hash || hash === '/') return '/';
    return hash.startsWith('/app') ? '/app/' : '/';
  };

  let route = $state(normalizeRoute());

  function navigate(target) {
    const normalized = target.startsWith('/') ? target : `/${target}`;
    const nextHash = normalized === '/' ? '#/' : `#${normalized}`;
    window.location.hash = nextHash;
  }

  onMount(() => {
    const updateRoute = () => {
      route = normalizeRoute(window.location.hash || '#/');
    };

    updateRoute();
    window.addEventListener('hashchange', updateRoute);
    return () => window.removeEventListener('hashchange', updateRoute);
  });
</script>

<App theme="ios" dark name="Ale or Fail">
  {#if route === '/'}
    <HomePage {navigate} />
  {:else}
    <AleOrFail {navigate} />
  {/if}
</App>
