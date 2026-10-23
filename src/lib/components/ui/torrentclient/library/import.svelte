<script lang='ts'>
  import MagnifyingGlass from 'svelte-radix/MagnifyingGlass.svelte'
  import { toast } from 'svelte-sonner'

  import { Badge } from '../../badge'
  import { Button } from '../../button'

  import { MediaCell } from './cells'

  import type { LibraryEntry } from 'native'

  import { Import } from '$lib/components/icons/animated'
  import { Checkbox } from '$lib/components/ui/checkbox'
  import * as Dialog from '$lib/components/ui/dialog'
  import { Input } from '$lib/components/ui/input'
  import { Label } from '$lib/components/ui/label'
  import { resolveNames } from '$lib/components/ui/player/resolver'
  import { Switch } from '$lib/components/ui/switch'
  import native from '$lib/modules/native'
  import { settings } from '$lib/modules/settings'
  import { server } from '$lib/modules/torrent'
  import { cn, fastPrettyBytes } from '$lib/utils'

  const { downloaded, library } = server

  let open = false
  let loading = false
  let entries: LibraryEntry[] = []
  let selected: Record<string, boolean> = {}
  let matches: Record<string, { mediaID: number, episode: number } | null> = {}
  const asked = new Set<string>()
  let filter = ''
  let move = false

  async function load () {
    loading = true
    selected = {}
    try {
      entries = (await native.qbittorrentLibrary()).sort((a, b) => b.date - a.date)
    } catch (error) {
      console.error(error)
      toast.error('Failed to read the qBittorrent library', { description: error instanceof Error ? error.message : 'Unknown error' })
    }
    loading = false
  }

  async function match (list: LibraryEntry[]) {
    const todo = list.filter(({ hash }) => !asked.has(hash))
    if (!todo.length) return
    for (const { hash } of todo) asked.add(hash)
    const resolved = await resolveNames(todo).catch(() => [])
    const next = { ...matches }
    for (const { hash } of todo) {
      const found = resolved.find(entry => entry.hash === hash)
      next[hash] = found ? { mediaID: found.metadata.media.id, episode: Number.parseInt(String(found.metadata.episode)) || 0 } : null
    }
    matches = next
  }

  $: if (open) load()
  $: if (!$settings.torrentPersist) move = false
  $: stored = new Map<string, LibraryEntry>($library.map(entry => [entry.hash, entry]))
  $: visible = entries.filter(({ name }) => name.toLowerCase().includes(filter.toLowerCase()))
  $: choices = visible.filter(({ hash }) => !$downloaded.has(hash))
  $: picked = Object.keys(selected).filter(hash => selected[hash])
  $: all = choices.length > 0 && choices.every(({ hash }) => selected[hash])
  $: match(choices)
  $: matching = picked.some(hash => !(hash in matches))

  function toggleAll () {
    const checked = !all
    for (const { hash } of choices) selected[hash] = checked
  }

  function importTorrents () {
    const torrents = picked.map(hash => ({ hash, mediaID: 0, episode: 0, ...matches[hash] }))
    toast.promise(
      native.importQBittorrent(torrents, move)
        .then(async imported => {
          await server.updateLibrary()
          return imported
        }), {
        loading: 'Importing torrents...',
        success: imported => `Imported ${imported.length} of ${torrents.length} torrents`,
        error: e => {
          console.error(e)
          return 'Failed to import torrents\n' + ('stack' in (e as object) ? (e as Error).stack : 'Unknown error')
        },
        description: 'This may take a VERY long while depending on the size of the torrents.'
      })
  }
</script>

<Dialog.Root portal='#root' bind:open>
  <Dialog.Trigger asChild let:builder>
    <Button variant='secondary' size='icon' class='border-0 animated-icon' builders={[builder]} aria-label='Import from qBittorrent'>
      <Import class='size-4' />
    </Button>
  </Dialog.Trigger>
  <Dialog.Content class='max-w-3xl flex flex-col'>
    <Dialog.Header>
      <Dialog.Title>Import from qBittorrent</Dialog.Title>
      <Dialog.Description>
        Select the torrents to add to your library. {move ? 'Their files are moved out of the qBittorrent folder.' : 'Their files are copied, so qBittorrent keeps its own.'}
      </Dialog.Description>
    </Dialog.Header>
    <div class='flex items-center scale-parent relative overflow-visible'>
      <Input
        class='pl-9 bg-background select:bg-accent select:text-accent-foreground shadow-sm no-scale placeholder:opacity-50'
        placeholder='Search by Torrent Name...'
        bind:value={filter} />
      <MagnifyingGlass class='h-4 w-4 shrink-0 opacity-50 absolute left-3 text-muted-foreground z-10 pointer-events-none' />
    </div>
    <div class='flex items-center gap-3 text-sm text-muted-foreground'>
      <Checkbox checked={all} disabled={!choices.length} onCheckedChange={toggleAll} aria-label='Select all' />
      <span>{picked.length} of {entries.length} torrent(s) selected.</span>
    </div>
    <div class='rounded-md border max-h-[50vh] overflow-y-auto'>
      {#each visible as entry (entry.hash)}
        {@const added = $downloaded.has(entry.hash)}
        {@const found = added ? stored.get(entry.hash) : matches[entry.hash]}
        <Label class={cn('flex items-start gap-3 px-3 py-2 border-b last:border-b-0', added ? 'opacity-50' : 'cursor-pointer select:bg-accent/50', selected[entry.hash] && 'bg-accent/30')}>
          {#if added}
            <Checkbox class='mt-1' checked disabled />
          {:else}
            <Checkbox class='mt-1' bind:checked={selected[entry.hash]} />
          {/if}
          <div class='min-w-0 grow'>
            {#if found}
              <MediaCell mediaID={found.mediaID} episode={found.episode} name={entry.name} />
            {:else}
              <div class='truncate font-medium' title={entry.name}>{entry.name}</div>
            {/if}
            <div class='mt-1 text-xs text-muted-foreground font-normal'>
              {fastPrettyBytes(entry.size)}, {Math.floor(entry.progress * 100)}% downloaded
            </div>
            {#if found === null}
              <div class='mt-2 text-xs text-muted-foreground font-normal'>No anime found</div>
            {:else if !found && !added}
              <div class='mt-2 flex items-center gap-3'>
                <div class='h-12 w-8 shrink-0 rounded bg-primary/10 animate-pulse' />
                <div class='flex flex-col gap-1.5'>
                  <div class='h-3.5 w-40 max-w-full bg-primary/10 animate-pulse rounded' />
                  <div class='h-3 w-24 bg-primary/10 animate-pulse rounded' />
                </div>
              </div>
            {/if}
          </div>
          {#if added}
            <Badge variant='secondary' class='shrink-0'>In library</Badge>
          {/if}
        </Label>
      {:else}
        {#if loading}
          {#each Array.from({ length: 3 }) as _, i (i)}
            <div class='flex items-start gap-3 px-3 py-2 border-b last:border-b-0'>
              <div class='mt-1 size-[18px] shrink-0 rounded-sm bg-primary/10 animate-pulse' />
              <div class='flex flex-col gap-1.5 grow'>
                <div class='h-3.5 w-3/5 bg-primary/10 animate-pulse rounded' />
                <div class='h-3 w-24 bg-primary/10 animate-pulse rounded' />
              </div>
            </div>
          {/each}
        {:else}
          <div class='h-24 flex items-center justify-center text-sm text-muted-foreground'>
            {entries.length ? 'No torrents match your search.' : 'No qBittorrent library found.'}
          </div>
        {/if}
      {/each}
    </div>
    <div class='flex items-center gap-2'>
      <Switch id='qbittorrent-move' hideState bind:checked={move} disabled={!$settings.torrentPersist} />
      <Label for='qbittorrent-move' class='text-sm'>Move the files instead of copying them</Label>
    </div>
    {#if !$settings.torrentPersist}
      <div class='text-xs text-muted-foreground'>
        Persist Files is off, so Hayase deletes an imported torrent and its files after you play another torrent. Turn Persist Files on in the client settings to keep them. Moving is disabled for this reason.
      </div>
    {/if}
    <Dialog.Footer>
      <Dialog.Close let:builder asChild>
        <Button builders={[builder]} disabled={!picked.length || matching} on:click={importTorrents}>
          Import
        </Button>
      </Dialog.Close>
      <Dialog.Close let:builder asChild>
        <Button variant='secondary' builders={[builder]}>Cancel</Button>
      </Dialog.Close>
    </Dialog.Footer>
  </Dialog.Content>
</Dialog.Root>
