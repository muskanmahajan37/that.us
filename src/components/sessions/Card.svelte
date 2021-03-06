<script>
  export let editMode = false;
  export let id;
  export let title;
  export let shortDescription;
  export let startTime;
  export let speakers;
  export let status;
  export let tags = [];
  export let attendees = []; // todo.. needs to be favorites
  export let __typename; // just here to clean up props
  export let eventId = config.eventId; // just here to clean up props
  export let type = 'OPEN_SPACE'; // just here to clean up props

  // 3rd party
  import { onMount } from 'svelte';
  import { Link } from 'yrv';
  import dayjs from 'dayjs';
  import isBetween from 'dayjs/plugin/isBetween';
  import Icon from 'svelte-awesome';
  import { info, heart, signIn, cog, caretDown } from 'svelte-awesome/icons';
  import qs from 'query-string';
  import { getClient } from '@urql/svelte';
  import _ from 'lodash';

  // supporting goo
  import config from '../../config';
  import { isAuthenticated, thatProfile } from '../../utilities/security';
  import { truncate, isLongerThan } from '../../utilities/truncate';
  import favoritesApi from '../../dataSources/api.that.tech/favorites';
  import { show } from '../../store/profileNotification';

  // UI Elements
  import { Tag } from '../../elements';
  import CardLink from './CardLink.svelte';

  dayjs.extend(isBetween);

  const { toggle, get: getFavorites, favoritesStore: favorites } = favoritesApi(
    getClient(),
  );
  let host = speakers[0];

  let imageCrop = '?mask=ellipse&w=500&h=500&fit=crop';
  let favoriteDisabled = false;
  let showExpandedDescription = false;

  const isAllowed = () => {
    let permitted = false;

    if (_.isEmpty($thatProfile)) {
      show.set(new Boolean(true));
      permitted = false;
    } else {
      permitted = true;
    }

    return permitted;
  };

  const handleToggle = async () => {
    if (isAllowed()) {
      favoriteDisabled = true;
      await toggle(id);
      favoriteDisabled = false;
    }
  };

  let expandDescription = false;

  let isFavorite = false;
  favorites.subscribe(favs => {
    let found = _.find(favs, i => i.id === id);

    found ? (isFavorite = true) : (isFavorite = false);
  });

  let isInWindow = false;
  $: canJoin = isInWindow;

  onMount(async () => {
    if ($isAuthenticated) await getFavorites();

    const interval = setInterval(() => {
      let inSession = dayjs().isBetween(
        dayjs(startTime).subtract(5, 'minute'),
        dayjs(startTime).add(1, 'hour'),
        'minute',
      );

      isInWindow = inSession;

      if (!inSession) {
        const { join } = qs.parse(location.search);
        if (join) isInWindow = true;
      }
    }, 1000);

    return () => {
      clearInterval(interval);
    };
  });

  let userProfileImage = host.profileImage
    ? `${host.profileImage}${imageCrop}`
    : config.defaultProfileImage;
</script>

<div class="w-full h-full flex flex-col">

  <div class="flex items-center p-3 space-x-3">
    <Link
      open
      href="https://www.thatconference.com/member/{host.profileSlug}"
      class="flex-shrink-0"
    >

      <span class="inline-block relative">
        <img
          class="w-15 h-15 rounded-full"
          src="{userProfileImage}"
          alt="{`${host.firstName} ${host.lastName}`}"
        />

        {#if host.earnedMeritBadges.length > 0}
          <span class="absolute bottom-0 left-0 block h-6 w-6">
            <img
              src="{host.earnedMeritBadges[0].image}"
              alt="{host.earnedMeritBadges[0].name}"
            />
          </span>
        {/if}
      </span>

    </Link>
    <div class="flex flex-col">
      <h3 class="text-gray-900 text-sm leading-5 font-medium break-words">
        {title}
      </h3>
      <h3 class="text-gray-400 text-sm leading-5 pt-2">
        {`${host.firstName} ${host.lastName}`}
      </h3>
    </div>
  </div>

  <div
    class="flex-grow p-3"
    class:cursor-pointer="{isLongerThan(shortDescription, 25)}"
    on:click|preventDefault="{() => (expandDescription = !expandDescription)}"
  >
    <p class="text-gray-500 text-sm leading-5 break-words">
      {#if expandDescription}
        <span>{shortDescription}</span>
      {:else}
        <div class:hover:text-gray-300="{isLongerThan(shortDescription, 25)}">
          <span>{truncate(shortDescription, 25)}</span>
          {#if isLongerThan(shortDescription, 25)}
            <Icon data="{caretDown}" class="ml-2" />
          {/if}
        </div>
      {/if}

    </p>
  </div>

  <div class="flex flex-wrap items-center justify-around space-x-3 p-3">
    {#each tags as t}
      <Tag>{t}</Tag>
    {/each}
  </div>

  <div class="flex-none border-t border-gray-200">
    <div class="-mt-px flex">
      <div class="w-0 flex-1 flex border-r border-gray-200">
        <CardLink href="/sessions/{id}" icon="{info}" text="{'More Details'}" />
      </div>
      {#if $isAuthenticated}
        {#if canJoin}
          <div class="-ml-px w-0 flex-1 flex">
            <CardLink href="/join/{id}" icon="{signIn}" text="{'Join In'}" />
          </div>
        {:else if editMode}
          <div class="-ml-px w-0 flex-1 flex">
            <Link
              href="/sessions/edit/{id}"
              class="relative w-0 flex-1 inline-flex items-center justify-center
              py-4 text-sm leading-5 text-gray-700 font-medium border
              border-transparent rounded-br-lg hover:text-gray-300
              focus:outline-none focus:shadow-outline-blue focus:border-blue-300
              focus:z-10 transition ease-in-out duration-150"
            >

              <Icon data="{cog}" class="w-5 h-5" />
              <span class="ml-3">Edit</span>
            </Link>
          </div>
        {:else}
          <div class="-ml-px w-0 flex-1 flex">
            <button
              on:click|preventDefault="{!favoriteDisabled && handleToggle}"
              class:text-red-500="{isFavorite}"
              class="relative w-0 flex-1 inline-flex items-center justify-center
              py-4 text-sm leading-5 text-gray-700 font-medium border
              border-transparent rounded-br-lg hover:text-gray-300
              focus:outline-none focus:shadow-outline-blue focus:border-blue-300
              focus:z-10 transition ease-in-out duration-150"
            >

              <Icon data="{heart}" class="w-5 h-5" />
              <span class="ml-3">Favorite</span>
            </button>
          </div>
        {/if}
      {/if}
    </div>
  </div>

</div>
