<script lang="ts">
  import Post from '$lib/components/lemmy/post/Post.svelte'
  import {
    AdjustmentsHorizontal,
    AtSymbol,
    ChartBar,
    EllipsisHorizontal,
    Envelope,
    Fire,
    Icon,
    NoSymbol,
    PencilSquare,
    ShieldCheck,
    ShieldExclamation,
  } from 'svelte-hero-icons'
  import CommentItem from '$lib/components/lemmy/comment/CommentItem.svelte'
  import Pageination from '$lib/components/ui/Pageination.svelte'
  import { page } from '$app/stores'
  import { goto } from '$app/navigation'
  import { isCommentView } from '$lib/lemmy/item.js'
  import { client, getClient } from '$lib/lemmy.js'
  import { isBlocked } from '$lib/lemmy/user.js'
  import { Menu, MenuButton, removeToast, toast } from 'mono-svelte'
  import UserLink from '$lib/components/lemmy/user/UserLink.svelte'
  import { profile } from '$lib/auth.js'
  import { ban, isAdmin } from '$lib/components/lemmy/moderation/moderation.js'
  import ShieldIcon from '$lib/components/lemmy/moderation/ShieldIcon.svelte'
  import { searchParam } from '$lib/util.js'
  import Placeholder from '$lib/components/ui/Placeholder.svelte'
  import { Button, Modal, Select } from 'mono-svelte'
  import PrivateMessageModal from '$lib/components/lemmy/modal/PrivateMessageModal.svelte'
  import Expandable from '$lib/components/ui/Expandable.svelte'
  import { communityLink } from '$lib/lemmy/generic.js'
  import ItemList from '$lib/components/lemmy/generic/ItemList.svelte'
  import { t } from '$lib/translations.js'
  import EntityHeader from '$lib/components/ui/EntityHeader.svelte'
  import { publishedToDate } from '$lib/components/util/date.js'
  import { formatRelativeDate } from '$lib/components/util/RelativeDate.svelte'

  export let data

  let blocking = false

  async function blockUser(block: number) {
    if (!$profile?.user || !$profile?.jwt) throw new Error('Unauthenticated')

    blocking = true
    try {
      const blocked = isBlocked($profile.user, block)

      await getClient().blockPerson({
        block: !blocked,
        person_id: block,
      })

      if (blocked) {
        const index = $profile.user.person_blocks
          .map((p) => p.target.id)
          .indexOf(block)
        $profile.user.person_blocks.splice(index, 1)
      }

      toast({
        content: blocked ? $t('toast.unblockUser') : $t('toast.blockUser'),
        type: 'success',
      })

      goto($page.url, {
        invalidateAll: true,
      })
    } catch (err) {
      toast({
        content: err as any,
        type: 'error',
      })
    }
    blocking = false
  }

  let messaging = false

  let purgingUser = false
  let purgeEnabled = false
  $: if (purgingUser) {
    purgeEnabled = false
    setTimeout(() => {
      purgeEnabled = true
    }, 3000)
  }

  async function purgeUser() {
    purgingUser = false
    const purgeToast = toast({ content: '', loading: true })

    try {
      await client().purgePerson({
        person_id: data.person_view.person.id,
      })
      removeToast(purgeToast)
      toast({ content: $t('toast.purgeUser'), type: 'success' })
    } catch (e) {
      toast({ content: e as any, type: 'error' })
    }
  }
</script>

<svelte:head>
  <title>{data.person_view.person.name}</title>
</svelte:head>

{#if $profile?.user}
  <PrivateMessageModal
    bind:user={data.person_view.person}
    bind:open={messaging}
  />
{/if}

{#if purgingUser}
  <Modal bind:open={purgingUser}>
    <svelte:fragment slot="title">Purging User</svelte:fragment>
    <p>
      Purging user <span class="font-bold">
        {data.person_view.person.name}
      </span>
    </p>
    <p>
      Are you sure you want to do this? (The button will enable in 3 seconds.)
    </p>
    <div class="flex flex-row gap-2">
      <Button size="lg" on:click={() => (purgingUser = false)} class="flex-1">
        Cancel
      </Button>
      <Button
        size="lg"
        color="danger"
        on:click={purgeUser}
        disabled={!purgeEnabled}
        class="flex-1"
      >
        Purge
      </Button>
    </div>
  </Modal>
{/if}

<div class="flex flex-col gap-4 max-w-full w-full">
  <div class="">
    <EntityHeader
      avatar={data.person_view.person.avatar}
      name={data.person_view.person.display_name ||
        data.person_view.person.name}
      banner={data.person_view.person.banner}
      bio={data.person_view.person.bio}
      stats={[
        {
          name: $t('content.posts'),
          value: data.person_view.counts.post_count.toString(),
        },
        {
          name: $t('content.comments'),
          value: data.person_view.counts.comment_count.toString(),
        },
        {
          name: $t('stats.joined'),
          value: formatRelativeDate(
            publishedToDate(data.person_view.person.published),
            { style: 'short' }
          ).toString(),
          format: false,
        },
      ]}
    >
      <span class="text-sm flex gap-0 items-center w-max" slot="nameDetail">
        @
        <UserLink
          showInstance
          user={data.person_view.person}
          displayName={false}
          class="font-normal"
        />
      </span>
      {#if (data.moderates ?? []).length > 0}
        <Expandable
          class="border rounded-xl bg-white/50 dark:bg-zinc-900/50 w-full p-3 px-4
      dark:border-zinc-800 border-slate-300 border-opacity-50 text-slate-700 dark:text-zinc-300 transition-colors"
        >
          <span slot="title" class="flex items-center gap-1">
            <ShieldIcon width={14} filled />
            {$t('routes.profile.moderates')}
          </span>
          <ItemList
            items={data.moderates.map((m) => ({
              id: m.community.id,
              name: m.community.title,
              url: communityLink(m.community),
              avatar: m.community.icon,
              instance: new URL(m.community.actor_id).hostname,
            }))}
          />
        </Expandable>
      {/if}
      <svelte:fragment slot="actions">
        {#if $profile?.user && $profile.jwt && data.person_view.person.id != $profile.user.local_user_view.person.id}
          <div class="flex items-center gap-2 w-full">
            <Button
              size="square-md"
              color="secondary"
              href="/inbox/messages/{data.person_view.person.id}"
              title="Message"
            >
              <Icon slot="prefix" solid size="16" src={Envelope} />
            </Button>
            {#if data.person_view.person.matrix_user_id}
              <Button
                size="square-md"
                color="secondary"
                href="https://matrix.to/#/{data.person_view.person
                  .matrix_user_id}"
                title="Matrix User"
              >
                <Icon slot="prefix" solid size="16" src={AtSymbol} />
              </Button>
            {/if}
            {#if isAdmin($profile?.user)}
              <Menu class="ml-auto" placement="bottom-end">
                <Button size="square-md" slot="target">
                  <ShieldIcon width={16} filled />
                </Button>
                <MenuButton
                  color="danger-subtle"
                  on:click={() =>
                    ban(
                      data.person_view.person.banned,
                      data.person_view.person
                    )}
                >
                  <Icon slot="prefix" mini size="16" src={ShieldExclamation} />
                  {data.person_view.person.banned ? 'Unban' : 'Ban'}
                </MenuButton>
                <MenuButton
                  color="danger-subtle"
                  on:click={() => (purgingUser = !purgingUser)}
                >
                  <Icon slot="prefix" mini size="16" src={Fire} />
                  Purge
                </MenuButton>
              </Menu>
            {/if}
            <Menu placement="bottom-end">
              <Button size="square-md" slot="target">
                <Icon src={EllipsisHorizontal} slot="prefix" size="16" mini />
              </Button>
              <MenuButton
                color="danger-subtle"
                on:click={() => blockUser(data.person_view.person.id)}
              >
                <Icon slot="prefix" mini size="16" src={NoSymbol} />
                {isBlocked($profile.user, data.person_view.person.id)
                  ? 'Unblock'
                  : 'Block'}
              </MenuButton>
            </Menu>
          </div>
        {/if}
      </svelte:fragment>
    </EntityHeader>
  </div>

  <div class="flex flex-col gap-4 max-w-full w-full min-w-0">
    <div class="flex flex-row gap-4 flex-wrap">
      <Select
        bind:value={data.type}
        on:change={() => searchParam($page.url, 'type', data.type, 'page')}
      >
        <span slot="label" class="flex items-center gap-1">
          <Icon src={AdjustmentsHorizontal} size="15" mini />
          {$t('filter.type')}
        </span>
        <option value="all">{$t('content.all')}</option>
        <option value="posts">{$t('content.posts')}</option>
        <option value="comments">{$t('content.comments')}</option>
      </Select>
      <Select
        bind:value={data.sort}
        on:change={() => searchParam($page.url, 'sort', data.sort, 'page')}
      >
        <span slot="label" class="flex items-center gap-1">
          <Icon src={ChartBar} size="14" mini />
          {$t('filter.sort.label')}
        </span>
        <option value="New">{$t('filter.sort.new')}</option>
        <option value="TopAll">{$t('filter.sort.top.label')}</option>
        <option value="Old">{$t('filter.sort.old')}</option>
      </Select>
    </div>
    {#if data.items.length == 0}
      <Placeholder
        icon={PencilSquare}
        title="No submissions"
        description="This user has no submissions that match this filter."
      />
    {:else}
      <div
        class="!divide-y divide-slate-200 dark:divide-zinc-800 flex flex-col"
      >
        {#each data.items as item}
          {#if isCommentView(item) && (data.type == 'all' || data.type == 'comments')}
            <CommentItem comment={item} />
          {:else if !isCommentView(item) && (data.type == 'all' || data.type == 'posts')}
            <Post post={item} />
          {/if}
        {/each}
      </div>
    {/if}
    <Pageination
      page={data.page}
      on:change={(p) => searchParam($page.url, 'page', p.detail.toString())}
    />
  </div>
</div>
