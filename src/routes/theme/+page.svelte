<script lang="ts">
  import {
    Button,
    Material,
    Modal,
    TextArea,
    action,
    modal,
    toast,
  } from 'mono-svelte'
  import { defaultColors, type Colors, colors } from '$lib/ui/colors'
  import {
    ArrowDownTray,
    ArrowPath,
    ArrowUpTray,
    Bookmark,
    Icon,
  } from 'svelte-hero-icons'
  import ColorSwatch from './ColorSwatch.svelte'
  import { t } from '$lib/translations'
  import Header from '$lib/components/ui/layout/pages/Header.svelte'

  const c = defaultColors

  const svelteIntellisenseSucks = (): (keyof Colors)[] =>
    ['slate', 'zinc', 'other'] as (keyof Colors)[]

  let importing = false
  let importText = ''
</script>

{#if importing}
  <Modal
    bind:open={importing}
    on:action={() => {
      try {
        if (importText == '') {
          throw new Error('Import is empty')
        }
        $colors = JSON.parse(importText)
        toast({ content: $t('message.success'), type: 'success' })
        importing = false
      } catch (err) {
        // @ts-ignore
        toast({ content: err, type: 'error' })
      }
    }}
    title={$t('routes.theme.import')}
    action={$t('routes.theme.import')}
  >
    <TextArea bind:value={importText} style="font-family: monospace;" />
  </Modal>
{/if}

<div class="flex flex-col gap-4 h-full">
  <Header>Theme</Header>
  <div class="flex items-center gap-4">
    <Button
      on:click={() => {
        importing = !importing
      }}
      size="lg"
    >
      <Icon src={ArrowUpTray} size="16" mini />
      {$t('routes.theme.import')}
    </Button>
    <Button
      on:click={() => {
        navigator.clipboard.writeText(JSON.stringify($colors))
        toast({ content: 'Copied theme to clipboard.' })
      }}
      size="lg"
    >
      <Icon src={ArrowDownTray} size="16" mini />
      {$t('routes.theme.export')}
    </Button>
    <Button
      on:click={() => {
        modal({
          actions: [
            action({
              content: $t('common.cancel'),
              close: true,
            }),
            action({
              action: () => {
                $colors = { other: {}, primary: {}, zinc: {}, slate: {} }
              },
              content: $t('routes.theme.reset'),
              close: true,
              type: 'danger',
            }),
          ],
          title: $t('routes.theme.resetWarning.title'),
          body: $t('routes.theme.resetWarning.description'),
        })
      }}
      size="lg"
    >
      <Icon src={ArrowPath} size="16" mini />
      {$t('routes.theme.reset')}
    </Button>
  </div>
  <Material color="transparent" class="items-center gap-x-4 color-grid gap-y-2">
    <h1 class="text-2xl font-bold col-span-2">{$t('routes.theme.accent')}</h1>
    <ColorSwatch
      backgroundColor={defaultColors.primary[900]}
      bind:value={$colors.primary[900]}
      class="!w-12 !h-12 col-span-1"
    />
    <ColorSwatch
      backgroundColor={defaultColors.primary[100]}
      bind:value={$colors.primary[100]}
      class="!w-12 !h-12 col-span-1"
    />
    <span class="font-semibold text-base">
      {$t('nav.menu.colorscheme.light')}
    </span>
    <span class="font-semibold text-base">
      {$t('nav.menu.colorscheme.dark')}
    </span>
  </Material>
  <div class="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-3 gap-2">
    {#each svelteIntellisenseSucks() as category}
      <Material color="transparent" class="flex flex-col gap-2">
        <h1 class="capitalize font-semibold text-lg">{category}</h1>
        <div class="flex flex-row gap-1 flex-wrap items-center space-evenly">
          {#each Object.keys(c[category]) as shade}
            <div class="flex flex-col gap-0.5 w-10 group">
              <!--@ts-ignore-->
              <ColorSwatch
                bind:value={$colors[category][shade]}
                backgroundColor={defaultColors[category][shade]}
              />
              <span class="font-medium capitalize">{shade}</span>
            </div>
          {/each}
        </div>
      </Material>
    {/each}
  </div>
  <div
    class="flex items-center gap-2 sm:gap-8 flex-col sm:flex-row mt-auto mx-auto text-sm text-slate-600 dark:text-zinc-400"
  >
    <span>Slate is for light theme, Zinc is for dark</span>
    <span>
      You can share themes at <a
        href="/c/photon@lemdro.id"
        class="text-sky-600 dark:text-sky-500 hover:underline"
      >
        photon@lemdro.id
      </a>
    </span>
  </div>
</div>

<style>
  :global(.color-grid) {
    display: grid;
    grid-template-columns: min-content min-content;
  }
</style>
