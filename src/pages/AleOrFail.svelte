<script>
  import { Sheet, Button } from 'framework7-svelte';
  import BrandLockup from '../components/BrandLockup.svelte';

  let { navigate } = $props();

  const storageKey = 'ale-or-fail-ratings-v1';

  function loadRatings() {
    if (typeof localStorage === 'undefined') return [];
    try {
      const saved = JSON.parse(localStorage.getItem(storageKey) || '[]');
      return Array.isArray(saved) ? saved : [];
    } catch {
      return [];
    }
  }

  function emptyForm() {
    return { name: '', brewery: '', description: '', rating: '', type: 'ale' };
  }

  let ratings = $state(loadRatings());
  let sheetOpen = $state(false);
  let activeTarget = $state(null);
  let editingId = $state(null);
  let form = $state(emptyForm());
  let breweryFilter = $state('all');
  let typeFilter = $state('all');
  let sortDirection = $state('desc');

  $effect(() => {
    try {
      localStorage.setItem(storageKey, JSON.stringify(ratings));
    } catch (error) {
      console.error('Failed to save ratings to localStorage:', error);
    }
  });

  const breweries = $derived.by(() => {
    const unique = new Set();
    for (const item of ratings) {
      const name = item.brewery?.trim();
      if (name) unique.add(name);
    }
    return Array.from(unique).sort((a, b) => a.localeCompare(b));
  });

  const filteredRatings = $derived(
    ratings.filter(
      (item) =>
        (breweryFilter === 'all' || item.brewery === breweryFilter) &&
        (typeFilter === 'all' || item.type === typeFilter),
    ),
  );

  const orderedRatings = $derived.by(() => {
    return [...filteredRatings].sort((a, b) => {
      const difference =
        sortDirection === 'desc' ? b.rating - a.rating : a.rating - b.rating;
      return (
        difference ||
        (sortDirection === 'desc'
          ? b.createdAt - a.createdAt
          : a.createdAt - b.createdAt)
      );
    });
  });

  function saveRatings(next) {
    ratings = next;
  }

  function openNew() {
    if (activeTarget !== 'new') {
      activeTarget = 'new';
      editingId = null;
      form = emptyForm();
    }
    sheetOpen = true;
  }

  function edit(item) {
    if (activeTarget !== item.id) {
      activeTarget = item.id;
      editingId = item.id;
      form = {
        name: item.name,
        brewery: item.brewery || '',
        description: item.description,
        rating: String(item.rating),
        type: item.type,
      };
    }
    sheetOpen = true;
  }

  function submit(event) {
    event?.preventDefault();
    const value = Number(form.rating);
    if (!form.name.trim() || !Number.isFinite(value) || value < 0 || value > 10)
      return;
    const item = {
      id: editingId || crypto.randomUUID(),
      name: form.name.trim(),
      brewery: form.brewery.trim(),
      description: form.description.trim(),
      rating: Math.round(value * 10) / 10,
      type: form.type,
      createdAt: editingId
        ? ratings.find((rating) => rating.id === editingId)?.createdAt || Date.now()
        : Date.now(),
    };
    saveRatings(
      editingId
        ? ratings.map((rating) => (rating.id === editingId ? item : rating))
        : [...ratings, item],
    );
    activeTarget = null;
    editingId = null;
    form = emptyForm();
    sheetOpen = false;
  }

  function remove(item) {
    if (confirm(`Delete ${item.name}?`)) {
      saveRatings(ratings.filter((rating) => rating.id !== item.id));
      if (activeTarget === item.id) {
        activeTarget = null;
        editingId = null;
        form = emptyForm();
      }
      sheetOpen = false;
    }
  }

  function ratingLabel(value) {
    return Number.isInteger(value) ? value.toFixed(0) : value.toFixed(1);
  }

  function returnHome() {
    if (typeof navigate === 'function') {
      navigate('/');
    } else {
      window.location.hash = '#/';
    }
  }

</script>

<main class="app-shell">
  <header class="app-header">
    <div class="brand-row">
      <BrandLockup />
      <button
        class="settings-button"
        type="button"
        onclick={returnHome}
        aria-label="Back to home"
      >
        <svg viewBox="0 0 24 24" aria-hidden="true">
          <path d="m6 6 12 12M18 6 6 18" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" />
        </svg>
      </button>
    </div>
  </header>
  <div class="page-header-meta">
    <p class="eyebrow">YOUR BEER LOG</p>
    <div class="header-meta">
      {#if ratings.length}
        <p class="summary">
          {ratings.length}
          {ratings.length === 1 ? 'rating' : 'ratings'} logged
        </p>
      {:else}
        <p class="summary">Your honest pint opinions, all in one place.</p>
      {/if}
    </div>
  </div>

  {#if ratings.length}
    <div class="filters" aria-label="Filter ratings">
      <label>
        Brewery
        <select bind:value={breweryFilter}>
          <option value="all">All breweries</option>
          {#each breweries as brewery}<option value={brewery}>{brewery}</option>{/each}
        </select>
      </label>
      <label>
        Type
        <select bind:value={typeFilter}>
          <option value="all">All types</option>
          <option value="ale">Ale</option>
          <option value="lager">Lager</option>
        </select>
      </label>
    </div>
  {/if}

  {#if orderedRatings.length}
    <button
      class="section-label sort-toggle"
      type="button"
      onclick={() =>
        (sortDirection = sortDirection === 'desc' ? 'asc' : 'desc')}
    >
      {sortDirection === 'desc' ? 'TOP TO BOTTOM' : 'BOTTOM TO TOP'}
      <span aria-hidden="true">⇅</span>
    </button>
    <div class="rating-list">
      {#each orderedRatings as item (item.id)}
        <button class="rating-card" onclick={() => edit(item)} type="button">
          <div class="rating-score">{ratingLabel(item.rating)}</div>
          <div class="rating-copy">
            <div class="name-row">
              <h2>{item.name}</h2>
              <span class:lager={item.type === 'lager'} class="type-pill">{item.type}</span>
            </div>
            <p>
              {item.brewery || 'Unknown brewery'}{item.description
                ? ` · ${item.description}`
                : ''}
            </p>
          </div>
          <span class="chevron">›</span>
        </button>
      {/each}
    </div>
  {:else if ratings.length}
    <div class="empty-state filtered-empty">
      <h2>No matching ratings</h2>
      <p>Try a different brewery or type.</p>
    </div>
  {:else}
    <div class="empty-state">
      <div class="beer-mark">✦</div>
      <h2>First round?</h2>
      <p>Add a beer you've tried and give it the score it deserves.</p>
    </div>
  {/if}

  <div class="action-dock">
    <button
      class="add-button"
      onclick={openNew}
      aria-label="Add a beer rating"><span>+</span> Add rating</button
    >
  </div>

  <Sheet
    opened={sheetOpen}
    onSheetClosed={() => (sheetOpen = false)}
    class="rating-sheet"
    backdrop
    swipeToClose={true}
    swipeHandler=".sheet-drag-area, .sheet-title-area"
  >
    <div class="sheet-drag-area" role="presentation">
      <div class="sheet-handle"></div>
    </div>
    <div class="sheet-content">
      <div class="sheet-heading">
        <div class="sheet-title-area">
          <p class="eyebrow">{editingId ? 'EDIT RATING' : 'NEW RATING'}</p>
          <h2>{editingId ? 'Make it right' : 'How was it?'}</h2>
        </div>
        <button
          class="close-button"
          onclick={() => (sheetOpen = false)}
          aria-label="Close">×</button
        >
      </div>
      <form onsubmit={submit}>
        <label
          >Name <input
            bind:value={form.name}
            placeholder="e.g. Oracle"
            maxlength="80"
            required
          /></label
        >
        <label
          >Brewery <input
            bind:value={form.brewery}
            placeholder="e.g. Salopian"
            maxlength="80"
          /></label
        >
        <label
          >Description <textarea
            bind:value={form.description}
            placeholder="Where, when, tasting notes…"
            maxlength="280"
            rows="3"
          ></textarea></label
        >
        <div class="form-row">
          <label
            >Rating <input
              bind:value={form.rating}
              type="number"
              min="0"
              max="10"
              step="0.1"
              inputmode="decimal"
              placeholder="0.0"
              required
            /></label
          >
          <fieldset>
            <legend>Type</legend>
            <div class="type-toggle">
              <button
                type="button"
                class:active={form.type === 'ale'}
                onclick={() => (form.type = 'ale')}>Ale</button
              >
              <button
                type="button"
                class:active={form.type === 'lager'}
                onclick={() => (form.type = 'lager')}>Lager</button
              >
            </div>
          </fieldset>
        </div>
        <Button fill large type="submit" class="save-button"
          >{editingId ? 'Save changes' : 'Save rating'}</Button
        >
        {#if editingId}<button
            type="button"
            class="delete-button"
            onclick={() => {
              const item = ratings.find((r) => r.id === editingId);
              if (item) remove(item);
            }}>Delete rating</button
          >{/if}
      </form>
    </div>
  </Sheet>

</main>
