<script>
  import { App, Sheet, Button } from 'framework7-svelte';
  import { onMount } from 'svelte';

  const storageKey = 'ale-or-fail-ratings-v1';
  let ratings = [];
  let sheetOpen = false;
  let editingId = null;
  let form = emptyForm();
  let breweryFilter = 'all';
  let typeFilter = 'all';
  let sortDirection = 'desc';
  let settingsOpen = false;
  let importInput;

  function emptyForm() {
    return { name: '', brewery: '', description: '', rating: '', type: 'ale' };
  }

  onMount(() => {
    try {
      const saved = JSON.parse(localStorage.getItem(storageKey) || '[]');
      ratings = Array.isArray(saved) ? saved : [];
    } catch {
      ratings = [];
    }
  });

  $: breweries = [...new Set(ratings.map((item) => item.brewery?.trim()).filter(Boolean))].sort((a, b) => a.localeCompare(b));
  $: filteredRatings = ratings.filter((item) =>
    (breweryFilter === 'all' || item.brewery === breweryFilter) &&
    (typeFilter === 'all' || item.type === typeFilter),
  );
  $: orderedRatings = [...filteredRatings].sort((a, b) => {
    const difference = sortDirection === 'desc' ? b.rating - a.rating : a.rating - b.rating;
    return difference || (sortDirection === 'desc' ? b.createdAt - a.createdAt : a.createdAt - b.createdAt);
  });

  function saveRatings(next) {
    ratings = next;
    localStorage.setItem(storageKey, JSON.stringify(next));
  }

  function openNew() {
    editingId = null;
    form = emptyForm();
    sheetOpen = true;
  }

  function edit(item) {
    editingId = item.id;
    form = { name: item.name, brewery: item.brewery || '', description: item.description, rating: String(item.rating), type: item.type };
    sheetOpen = true;
  }

  function submit() {
    const value = Number(form.rating);
    if (!form.name.trim() || !Number.isFinite(value) || value < 0 || value > 10) return;
    const item = {
      id: editingId || crypto.randomUUID(),
      name: form.name.trim(),
      brewery: form.brewery.trim(),
      description: form.description.trim(),
      rating: Math.round(value * 10) / 10,
      type: form.type,
      createdAt: editingId ? ratings.find((rating) => rating.id === editingId)?.createdAt || Date.now() : Date.now(),
    };
    saveRatings(editingId ? ratings.map((rating) => rating.id === editingId ? item : rating) : [...ratings, item]);
    sheetOpen = false;
  }

  function remove(item) {
    if (confirm(`Delete ${item.name}?`)) saveRatings(ratings.filter((rating) => rating.id !== item.id));
  }

  function ratingLabel(value) {
    return Number.isInteger(value) ? value.toFixed(0) : value.toFixed(1);
  }

  async function exportBackup() {
    const backup = {
      app: 'Ale or Fail',
      version: 1,
      exportedAt: new Date().toISOString(),
      ratings,
    };
    const blob = new Blob([JSON.stringify(backup, null, 2)], { type: 'application/json' });
    const date = new Date().toISOString().slice(0, 10);
    const filename = `ale-or-fail-backup-${date}.json`;
    const file = new File([blob], filename, { type: 'application/json' });

    if ('showSaveFilePicker' in window) {
      try {
        const fileHandle = await window.showSaveFilePicker({
          suggestedName: filename,
          types: [{ description: 'JSON backup', accept: { 'application/json': ['.json'] } }],
        });
        const writable = await fileHandle.createWritable();
        await writable.write(blob);
        await writable.close();
        return;
      } catch (error) {
        if (error.name === 'AbortError') return;
      }
    }

    if (navigator.canShare?.({ files: [file] })) {
      try {
        await navigator.share({ title: 'Ale or Fail backup', files: [file] });
        return;
      } catch (error) {
        if (error.name === 'AbortError') return;
      }
    }

    const downloadUrl = URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.href = downloadUrl;
    link.download = filename;
    document.body.appendChild(link);
    link.click();
    link.remove();
    setTimeout(() => URL.revokeObjectURL(downloadUrl), 1000);
  }

  function openImportPicker() {
    importInput?.click();
  }

  async function importBackup(event) {
    const file = event.currentTarget.files?.[0];
    if (!file) return;

    try {
      const backup = JSON.parse(await file.text());
      if (!backup || !Array.isArray(backup.ratings)) throw new Error('Not an Ale or Fail backup file.');

      const importedRatings = backup.ratings.map((item) => {
        const rating = Number(item?.rating);
        if (!item || !item.name || !Number.isFinite(rating) || rating < 0 || rating > 10) {
          throw new Error('The backup contains an invalid rating.');
        }
        return {
          id: typeof item.id === 'string' ? item.id : crypto.randomUUID(),
          name: String(item.name).trim(),
          brewery: typeof item.brewery === 'string' ? item.brewery.trim() : '',
          description: typeof item.description === 'string' ? item.description.trim() : '',
          rating: Math.round(rating * 10) / 10,
          type: item.type === 'lager' ? 'lager' : 'ale',
          createdAt: Number.isFinite(Number(item.createdAt)) ? Number(item.createdAt) : Date.now(),
        };
      });

      if (!confirm(`Replace your ${ratings.length} current ${ratings.length === 1 ? 'rating' : 'ratings'} with ${importedRatings.length} imported ${importedRatings.length === 1 ? 'rating' : 'ratings'}?`)) return;
      saveRatings(importedRatings);
      breweryFilter = 'all';
      typeFilter = 'all';
      settingsOpen = false;
      alert(`Imported ${importedRatings.length} ${importedRatings.length === 1 ? 'rating' : 'ratings'}.`);
    } catch (error) {
      alert(error.message || 'Unable to import that backup.');
    } finally {
      event.currentTarget.value = '';
    }
  }
</script>

<App theme="ios" dark name="Ale or Fail">
  <main class="app-shell">
      <header class="app-header">
        <div class="brand-row">
          <div class="brand-lockup">
            <svg class="beer-logo" viewBox="0 0 72 72" role="img" aria-label="Traditional foaming beer stein">
              <path d="M49 23h7c11 0 13 9 11 17-2 9-7 14-19 14" fill="none" stroke="#d9d8d4" stroke-width="7" stroke-linecap="round"/>
              <path d="M49 23h7c11 0 13 9 11 17-2 9-7 14-19 14" fill="none" stroke="#251b17" stroke-width="2" stroke-linecap="round"/>
              <path d="M14 20h37l-3 39c-.3 4-3 6-7 6H24c-4 0-6.7-2-7-6z" fill="#f5ad3f" stroke="#251b17" stroke-width="3" stroke-linejoin="round"/>
              <path d="M18 29c8 3 23 4 30 0l-1.2 14c-9 3-19 3-30 0z" fill="#d87a1d" opacity=".72"/>
              <path d="M18 47c8 3 20 3 28 0M21 54c6 2 16 2 23 0" fill="none" stroke="#7d3d13" stroke-width="2.2" stroke-linecap="round" opacity=".8"/>
              <path d="M17 22c-3-3.7-1.7-8.2 3-9.8 1.3-5.2 8.3-6.5 12.2-3.8 4-3.6 10.5-1.7 11.8 3.2 5.2-.8 9 4.5 6.7 8.7 3.2 3.2.8 7.8-3.4 7.4H19.5c-4.5.2-5.4-3.7-2.5-5.7z" fill="#fff2ca" stroke="#251b17" stroke-width="3" stroke-linejoin="round"/>
              <path d="M23 18c3-2 7-2.5 10-1M38 15c3 .2 5.6 1.4 7.5 3.8M17 24c10-2 23-2 34 .2" fill="none" stroke="#d6b67d" stroke-width="1.7" stroke-linecap="round"/>
              <path d="M23 34h20M22 39h21M23 44h19" stroke="#fbd17c" stroke-width="1.3" stroke-linecap="round" opacity=".8"/>
              <path d="M28 31v23M37 31v23" stroke="#9b4c16" stroke-width="1.3" opacity=".55"/>
            </svg>
            <h1>Ale or Fail</h1>
          </div>
          <button class="settings-button" type="button" on:click={() => settingsOpen = true} aria-label="Open settings">
            <svg viewBox="0 0 24 24" aria-hidden="true"><path d="M12 8a4 4 0 1 0 0 8 4 4 0 0 0 0-8Zm0-5 1.2 2.3c.7.2 1.4.5 2 .8l2.5-.8 2.1 2.1-.8 2.5c.4.6.6 1.3.8 2L22 12l-2.3 1.2c-.2.7-.5 1.4-.8 2l.8 2.5-2.1 2.1-2.5-.8c-.6.4-1.3.6-2 .8L12 22l-1.2-2.3c-.7-.2-1.4-.5-2-.8l-2.5.8-2.1-2.1.8-2.5c-.4-.6-.6-1.3-.8-2L2 12l2.3-1.2c.2-.7.5-1.4.8-2l-.8-2.5 2.1-2.1 2.5.8c.6-.4 1.3-.6 2-.8z" fill="none" stroke="currentColor" stroke-width="1.7" stroke-linejoin="round"/></svg>
          </button>
        </div>
        <p class="eyebrow">YOUR BEER LOG</p>
        <div class="header-meta">
          {#if ratings.length}
            <p class="summary">{ratings.length} {ratings.length === 1 ? 'rating' : 'ratings'} logged</p>
          {:else}
            <p class="summary">Your honest pint opinions, all in one place.</p>
          {/if}
        </div>
      </header>

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
        <button class="section-label sort-toggle" type="button" on:click={() => sortDirection = sortDirection === 'desc' ? 'asc' : 'desc'}>
          {sortDirection === 'desc' ? 'TOP TO BOTTOM' : 'BOTTOM TO TOP'} <span aria-hidden="true">⇅</span>
        </button>
        <div class="rating-list">
          {#each orderedRatings as item (item.id)}
            <button class="rating-card" on:click={() => edit(item)} type="button">
              <div class="rating-score">{ratingLabel(item.rating)}</div>
              <div class="rating-copy">
                <div class="name-row">
                  <h2>{item.name}</h2>
                  <span class:lager={item.type === 'lager'} class="type-pill">{item.type}</span>
                </div>
                <p>{item.brewery || 'Unknown brewery'}{item.description ? ` · ${item.description}` : ''}</p>
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
          <p>Add a beer you’ve tried and give it the score it deserves.</p>
        </div>
      {/if}

      <div class="action-dock">
        <button class="add-button" on:click={openNew} aria-label="Add a beer rating"><span>+</span> Add rating</button>
      </div>
  </main>

  <Sheet opened={sheetOpen} onSheetClosed={() => sheetOpen = false} class="rating-sheet" backdrop>
    <div class="sheet-handle"></div>
    <div class="sheet-content">
      <div class="sheet-heading">
        <div>
          <p class="eyebrow">{editingId ? 'EDIT RATING' : 'NEW RATING'}</p>
          <h2>{editingId ? 'Make it right' : 'How was it?'}</h2>
        </div>
        <button class="close-button" on:click={() => sheetOpen = false} aria-label="Close">×</button>
      </div>
      <form on:submit|preventDefault={submit}>
        <label>Name <input bind:value={form.name} placeholder="e.g. Oracle" maxlength="80" required /></label>
        <label>Brewery <input bind:value={form.brewery} placeholder="e.g. Salopian" maxlength="80" /></label>
        <label>Description <textarea bind:value={form.description} placeholder="Where, when, tasting notes…" maxlength="280" rows="3"></textarea></label>
        <div class="form-row">
          <label>Rating <input bind:value={form.rating} type="number" min="0" max="10" step="0.1" inputmode="decimal" placeholder="0.0" required /></label>
          <fieldset>
            <legend>Type</legend>
            <div class="type-toggle">
              <button type="button" class:active={form.type === 'ale'} on:click={() => form.type = 'ale'}>Ale</button>
              <button type="button" class:active={form.type === 'lager'} on:click={() => form.type = 'lager'}>Lager</button>
            </div>
          </fieldset>
        </div>
        <Button fill large type="submit" class="save-button">{editingId ? 'Save changes' : 'Save rating'}</Button>
        {#if editingId}<button type="button" class="delete-button" on:click={() => { const item = ratings.find((r) => r.id === editingId); if (item) remove(item); sheetOpen = false; }}>Delete rating</button>{/if}
      </form>
    </div>
  </Sheet>

  <Sheet opened={settingsOpen} onSheetClosed={() => settingsOpen = false} class="settings-sheet" backdrop>
    <div class="sheet-handle"></div>
    <div class="settings-content">
      <p class="eyebrow">SETTINGS</p>
      <h2>Your data</h2>
      <button class="settings-item" type="button" on:click={async () => { await exportBackup(); settingsOpen = false; }} disabled={!ratings.length}>
        <span>Export backup</span><span aria-hidden="true">›</span>
      </button>
      <button class="settings-item" type="button" on:click={openImportPicker}>
        <span>Import backup</span><span aria-hidden="true">›</span>
      </button>
    </div>
  </Sheet>
  <input class="import-input" bind:this={importInput} type="file" accept="application/json,.json" on:change={importBackup} />
</App>
