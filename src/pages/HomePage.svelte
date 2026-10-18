<script>
  import { Button, Card, Page, Sheet } from 'framework7-svelte';
  import BrandLockup from '../components/BrandLockup.svelte';

  export let navigate;
  let homeCardOpening = false;

  const storageKey = 'ale-or-fail-ratings-v1';
  let settingsOpen = false;
  let importInput;
  let ratingsCount = readRatings().length;

  function readRatings() {
    try {
      const saved = JSON.parse(localStorage.getItem(storageKey) || '[]');
      return Array.isArray(saved) ? saved : [];
    } catch {
      return [];
    }
  }

  async function exportBackup() {
    const backup = {
      app: 'Ale or Fail',
      version: 1,
      exportedAt: new Date().toISOString(),
      ratings: readRatings(),
    };
    const blob = new Blob([JSON.stringify(backup, null, 2)], {
      type: 'application/json',
    });
    const date = new Date().toISOString().slice(0, 10);
    const filename = `ale-or-fail-backup-${date}.json`;
    const file = new File([blob], filename, { type: 'application/json' });

    if ('showSaveFilePicker' in window) {
      try {
        const handle = await window.showSaveFilePicker({
          suggestedName: filename,
          types: [{ description: 'JSON backup', accept: { 'application/json': ['.json'] } }],
        });
        const writable = await handle.createWritable();
        await writable.write(blob);
        await writable.close();
        return;
      } catch (error) {
        if (error.name === 'AbortError') return;
      }
    }

    const url = URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.href = url;
    link.download = filename;
    document.body.appendChild(link);
    link.click();
    link.remove();
    setTimeout(() => URL.revokeObjectURL(url), 1000);
  }

  function openImportPicker() {
    importInput?.click();
  }

  async function importBackup(event) {
    const file = event.currentTarget.files?.[0];
    if (!file) return;

    try {
      const backup = JSON.parse(await file.text());
      if (!backup || !Array.isArray(backup.ratings)) {
        throw new Error('Not an Ale or Fail backup file.');
      }

      const imported = backup.ratings.map((item) => {
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

      if (!confirm(`Replace your ${ratingsCount} current ${ratingsCount === 1 ? 'rating' : 'ratings'} with ${imported.length} imported ${imported.length === 1 ? 'rating' : 'ratings'}?`)) return;
      localStorage.setItem(storageKey, JSON.stringify(imported));
      ratingsCount = imported.length;
      settingsOpen = false;
      alert(`Imported ${imported.length} ${imported.length === 1 ? 'rating' : 'ratings'}.`);
    } catch (error) {
      alert(error.message || 'Unable to import that backup.');
    } finally {
      event.currentTarget.value = '';
    }
  }

  function handleCardOpen() {
    homeCardOpening = true;
  }

  function handleCardClose() {
    homeCardOpening = false;
  }

  function handleCardOpened() {
    if (typeof navigate === 'function') {
      navigate('/app/');
    } else {
      window.location.hash = '#/app/';
    }
  }
</script>

<Page class="home-page" name="home">
  <button
    class="settings-button home-settings-button"
    type="button"
    onclick={() => (settingsOpen = true)}
    aria-label="Open settings"
  >
    <svg viewBox="0 0 24 24" aria-hidden="true">
      <path d="M12 8a4 4 0 1 0 0 8 4 4 0 0 0 0-8Zm0-5 1.2 2.3c.7.2 1.4.5 2 .8l2.5-.8 2.1 2.1-.8 2.5c.4.6.6 1.3.8 2L22 12l-2.3 1.2c-.2.7-.5 1.4-.8 2l.8 2.5-2.1 2.1-2.5-.8c-.6.4-1.3.6-2 .8L12 22l-1.2-2.3c-.7-.2-1.4-.5-2-.8l-2.5.8-2.1-2.1.8-2.5c-.4-.6-.6-1.3-.8-2L2 12l2.3-1.2c.2-.7.5-1.3.8-2l-.8-2.5 2.1-2.1 2.5.8c.6.4 1.3.6 2 .8z" fill="none" stroke="currentColor" stroke-width="1.7" stroke-linejoin="round" />
    </svg>
  </button>
  <div class="home-page-inner">
    <Card
      class="home-card"
      expandable
      expandableAnimateWidth
      onCardOpen={handleCardOpen}
      onCardOpened={handleCardOpened}
      onCardClose={handleCardClose}
      onCardClosed={handleCardClose}
    >
      {#snippet content()}
        <header
          class="app-header home-card-header"
          class:card-opening={homeCardOpening}
        >
          <div class="brand-row">
            <BrandLockup />
          </div>
        </header>
      {/snippet}
    </Card>
  </div>

  <Sheet
    opened={settingsOpen}
    onSheetClosed={() => (settingsOpen = false)}
    class="settings-sheet"
    backdrop
    swipeToClose={true}
    swipeHandler=".sheet-drag-area, .sheet-title-area"
  >
    <div class="sheet-drag-area" role="presentation">
      <div class="sheet-handle"></div>
    </div>
    <div class="settings-content">
      <div class="sheet-title-area">
        <p class="eyebrow">SETTINGS</p>
        <h2>Your data</h2>
      </div>
      <button
        class="settings-item"
        type="button"
        onclick={async () => {
          await exportBackup();
          settingsOpen = false;
        }}
        disabled={!ratingsCount}
      >
        <span>Export backup</span><span aria-hidden="true">›</span>
      </button>
      <button class="settings-item" type="button" onclick={openImportPicker}>
        <span>Import backup</span><span aria-hidden="true">›</span>
      </button>
    </div>
  </Sheet>
  <input
    class="import-input"
    bind:this={importInput}
    type="file"
    accept="application/json,.json"
    onchange={importBackup}
  />
</Page>
