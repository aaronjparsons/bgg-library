<script lang="ts">
    import dayjs from 'dayjs';
    import posthog from 'posthog-js'
    import type { ModalSettings, ModalComponent, ToastSettings } from '@skeletonlabs/skeleton';
    import { RadioGroup, RadioItem, modalStore, toastStore, ProgressBar } from '@skeletonlabs/skeleton';
    import { ViewListIcon, ViewGridIcon } from '@rgossiaux/svelte-heroicons/solid';
    import { onDestroy } from 'svelte';
    import { fade } from 'svelte/transition';
    import { writable, type Writable } from 'svelte/store';
    import { page } from '$app/stores';
    import { goto } from '$app/navigation';
    import logo from '$lib/assets/purple_meeple_150.png';
    import Dice from '$lib/icons/dice.png';
    import GameCard from "$lib/components/GameCard.svelte";
    import GameRow from "$lib/components/GameRow.svelte";
    import OptionsModal from "$lib/components/OptionsModal.svelte";
    import QRModal from "$lib/components/QRModal.svelte";
    import RandomGameModal from "$lib/components/RandomGameModal.svelte";
    import { getValue, sleep, getGameName, parseBestPlayerCount } from "$lib/utils";
    import { Library, libraryOptions, ratingKey } from "$lib/store";

    const username = $page.params.username;
    let displayType = 'grid';
    let displayName = '';
    let collection: Game[] = [];
    let collectionLoadAttempts = 0;
    let collectionLength = 0;
    let currentChunkRange = '';
    let chunkSize = 20;
    let currentChunkLoop = 0;
    let loadingState: string|null = null;

    const month = dayjs().month();
    const year = month === 0 ? dayjs().year() - 1 : dayjs().year();

    $: gameLoadingProgress = ((chunkSize * currentChunkLoop) / collectionLength) * 100;

    const setDisplayName = () => {
        const lastLetter = username.charAt(username.length - 1);
        displayName = lastLetter.toLowerCase() === 's' ? `${username}'` : `${username}'s`;
    }

    const setLibraryOptions = () => {
        const searchParams = new URLSearchParams(window.location.search);

        // Send analytics event if qrRef is included in query params
        if (searchParams.has('qrRef')) {
            posthog.capture('qr_ref', {
                username: username
            });
            searchParams.delete('qrRef');
        }

        const filters = ['playtime', 'playerCount', 'bestPlayerCount', 'rating', 'weight', 'played'];
        searchParams.forEach((value: string, key: string) => {
            if (filters.includes(key)) {
                $libraryOptions.filters[key] = value;
            } else {
                $libraryOptions[key] = key === 'includeExpansions' || key === 'useGeekRating'
                    ? true
                    : value;
            }
        })
    }

    const stripLeadingArticle = (string: string) => {
        return string.replace(/^(a |an |the )/i, '').trim();
    }

    const handleSort = (col: Game[]) => {
        let sorted: Game[] = [];
        const isAsc = $libraryOptions.sort === 'asc';

        switch($libraryOptions.selectedSort) {
            case 'release':
                sorted = col.sort((a: Game, b: Game) => {
                    const releasedA = parseInt(getValue(a.yearpublished));
                    const releasedB = parseInt(getValue(b.yearpublished));
                    return isAsc
                        ? releasedA > releasedB ? -1 : 1
                        : releasedA > releasedB ? 1 : -1
                });
                break;
            case 'rating':
                sorted = col.sort((a: Game, b: Game) => {
                    const ratingA = getValue(a.statistics.ratings[$ratingKey]);
                    const ratingB = getValue(b.statistics.ratings[$ratingKey]);
                    return isAsc
                        ? ratingA > ratingB ? -1 : 1
                        : ratingA > ratingB ? 1 : -1
                });
                break;
            case 'weight':
                sorted = col.sort((a: Game, b: Game) => {
                    const ratingA = getValue(a.statistics.ratings.averageweight);
                    const ratingB = getValue(b.statistics.ratings.averageweight);
                    return isAsc
                        ? ratingA > ratingB ? -1 : 1
                        : ratingA > ratingB ? 1 : -1
                });
                break;
            case 'alphabetical':
            default:
                sorted = col.sort((a: Game, b: Game) => {
                    const nameA = stripLeadingArticle(getGameName(a));
                    const nameB = stripLeadingArticle(getGameName(b));
                    return isAsc
                        ? nameA.localeCompare(nameB)
                        : nameB.localeCompare(nameA)
                });
                break;
        }

        return sorted;
    }

    const applyFilters = (col: Game[]) => {
        col = col.filter((game: Game) => {
            if (!$libraryOptions.includeExpansions && game['@_type'] !== 'boardgame') {
                return false;
            }

            if ($libraryOptions.filters.playerCount !== 'any') {
                const playerCount = parseInt($libraryOptions.filters.playerCount);
                const maxplayerCount = parseInt(getValue(game.maxplayers));
                const minplayerCount = parseInt(getValue(game.minplayers));
                if (!(playerCount >= minplayerCount && playerCount <= maxplayerCount)) {
                    return false;
                }
            }
            if ($libraryOptions.filters.bestPlayerCount !== 'any') {
                const selectedBestPlayerCount = parseInt($libraryOptions.filters.bestPlayerCount);
                const bestPlayerCounts = parseBestPlayerCount(game);

                if (bestPlayerCounts === '-') {
                    return false;
                }

                if (selectedBestPlayerCount === 9) {
                    // This is actually 9+. Consider all values 9 & above
                    if (Math.max(...bestPlayerCounts.split(',').map(c => parseInt(c))) < 9) {
                        return false;
                    }
                } else {
                    if (!bestPlayerCounts.includes(selectedBestPlayerCount)) {
                        return false;
                    }
                }
            }
            if ($libraryOptions.filters.playtime !== 'any') {
                const playtime = parseInt($libraryOptions.filters.playtime);
                const maxplaytime = parseInt(getValue(game.maxplaytime));
                if (maxplaytime > playtime) {
                    return false;
                }
            }
            if ($libraryOptions.filters.rating !== 'any') {
                const rating = parseInt($libraryOptions.filters.rating);
                const gameRating = parseFloat(getValue(game.statistics.ratings[$ratingKey]))

                if (gameRating < rating) {
                    return false;
                }
            }
            if ($libraryOptions.filters.weight !== 'any') {
                const weight = parseInt($libraryOptions.filters.weight);
                const avgweight = parseFloat(getValue(game.statistics.ratings.averageweight));
                if (avgweight > weight) {
                    return false;
                }
            }
            if ($libraryOptions.filters.played !== 'all') {
                const played = $libraryOptions.filters.played === 'true';

                if (played) {
                    return game.numplays > 0;
                } else {
                    return game.numplays === 0;
                }
            }

            return true;
        })

        return col;
    }

    const openRandomGame = () => {
        const modalComponent: ModalComponent = {
            // Pass a reference to your custom component
            ref: RandomGameModal,
            // Add your props as key/value pairs
            props: { currentGameList: collection },
        };
        const d: ModalSettings = {
            type: 'component',
            component: modalComponent
        };
        modalStore.trigger(d);

        posthog.capture('random_game_modal_opened');
    }

    const openQR = () => {
        const modalComponent: ModalComponent = {
            // Pass a reference to your custom component
            ref: QRModal,
            // Add your props as key/value pairs
            props: {  },
        };
        const d: ModalSettings = {
            type: 'component',
            component: modalComponent
        };
        modalStore.trigger(d);

        posthog.capture('qr_modal_opened');
    }

    const openOptions = () => {
        const modalComponent: ModalComponent = {
            // Pass a reference to your custom component
            ref: OptionsModal,
            // Add your props as key/value pairs
            props: {  },
        };
        const d: ModalSettings = {
            type: 'component',
            component: modalComponent,
            response: applyOptions
        };
        modalStore.trigger(d);

        posthog.capture('options_modal_opened');
    }

    const sortAndFilter = (arr: Game[]) => {
        let copy = [...arr];
        copy = applyFilters(copy);
        copy = handleSort(copy);
        return copy;
    }

    const applyOptions = () => {
        collection = sortAndFilter($Library.data);
        setSearchParams();
    }

    const fetchCollection = async () => {
        setDisplayName();

        // Set options based on search params
        setLibraryOptions();

        collectionLoadAttempts = 0;
        loadingState = 'collection';
        let requestingCollection = true;
        let attempts = 0;
        let gameIds = [];
        let coll = [];
        let updateRequired = false;
        while (requestingCollection) {
            const response = await fetch(`/api/collection?username=${username}`);
            // TODO: set updateRequired indicator & initiate a request retry
            if (response.ok) {
                if (response.status === 202) {
                    // BGG preparing
                    await sleep(10000);
                    collectionLoadAttempts++;
                } else {
                    const res = await response.json();
                    gameIds = Object.keys(res.collection);
                    coll = res.collection;
                    updateRequired = res.updateRequired;
                    requestingCollection = false;
                    if (updateRequired) {
                        backgroundUpdateRefresh();
                    }
                }
            } else {
                const { message } = await response.json();
                if (response.status === 404) {
                    // User doesn't exist
                    const toast: ToastSettings = {
                        message: 'Unable to find a user with that username. Please try again',
                        background: 'variant-filled-primary',
                        autohide: true,
                        timeout: 5000
                    };
                    toastStore.trigger(toast);
                    goto('/');
                    return;
                }

                return Promise.reject({ status: response.status, message })
            }
        }

        let collectionChunks = [];
        collectionLength = gameIds.length;
        loadingState = 'games';

        posthog.identify(username, { username });

        // Request details on items in collection
        for (let i = 0; i < collectionLength; i += chunkSize) {
            const currentChunk = gameIds.slice(i, i + chunkSize);

            const tail = i + chunkSize > collectionLength ? collectionLength : i + chunkSize;
            currentChunkRange = `${i + 1} - ${tail}`;

            const response = await fetch('/api/games', {
                method: 'POST',
                body: JSON.stringify({ gameIds: currentChunk, username }),
                headers: {
                    'content-type': 'application/json'
                }
            });

            if (response.ok) {
                const res = await response.json();
                for (const g of res.games) {
                    const id = g['@_id'];
                    const game = { ...g, ...coll[id] };
                    collectionChunks.push(game);
                }
                // Rate limit & let progress bar animate before next chunk
                await sleep(1750);
                currentChunkLoop++;
                await sleep(150);
            } else {
                const { message } = await response.json();
                return Promise.reject({ status: response.status, message });
            }
        }

        // Get counts
        const expansionCount = collectionChunks.filter(g => g['@_type'] === 'boardgameexpansion').length;
        const gameCount = collectionChunks.length - expansionCount;

        loadingState = null;
        $Library = {
            data: collectionChunks,
            username,
            updateRequired,
            loaded: true,
            gameCount,
            expansionCount
        };
        collection = sortAndFilter(collectionChunks);
        setSearchParams();
        await sleep(150);
    }

    const backgroundUpdateRefresh = async () => {
        // Wait for first re-attempt
        await sleep(15000);

        // Lazy copy paste of main fetch
        let updatingBackground = true;
        let gameIds = [];
        let coll = [];
        while (updatingBackground) {
            const response = await fetch(`/api/collection?username=${username}`);
            if (response.ok) {
                if (response.status === 202) {
                    // BGG still preparing
                    await sleep(20000);
                } else {
                    const res = await response.json();
                    const updateRequired = res.updateRequired;
                    if (updateRequired) {
                        // BGG still preparing
                        await sleep(20000);
                    } else {
                        gameIds = Object.keys(res.collection);
                        coll = res.collection;
                        updatingBackground = false;
                    }
                }
            } else {
                const { message } = await response.json();
                return Promise.reject({ status: response.status, message })
            }
        }

        let collectionChunks = [];
        collectionLength = gameIds.length;

        // Request details on items in collection
        for (let i = 0; i < collectionLength; i += (chunkSize + 1)) {
            const currentChunk = gameIds.slice(i, i + chunkSize + 1);

            const tail = i + chunkSize > collectionLength ? collectionLength : i + chunkSize;
            currentChunkRange = `${i || 1} - ${tail}`;

            const response = await fetch('/api/games', {
                method: 'POST',
                body: JSON.stringify({ gameIds: currentChunk, username }),
                headers: {
                    'content-type': 'application/json'
                }
            });

            if (response.ok) {
                const res = await response.json();
                for (const g of res.games) {
                    const id = g['@_id'];
                    const game = { ...g, ...coll[id] };
                    collectionChunks.push(game);
                }
                await sleep(1500);
            } else {
                const { message } = await response.json();
                return Promise.reject({ status: response.status, message })
            }
        }

        $Library = {
            data: collectionChunks,
            username,
            updateRequired: false,
            loaded: true
        };
        collection = sortAndFilter(collectionChunks);

        const toast: ToastSettings = {
            message: 'Collection updated',
            background: 'variant-filled-secondary',
            autohide: true,
            timeout: 3500
        };
        toastStore.trigger(toast);
    }

    const initFetchCollection = () => {
        collectionRequest = fetchCollection();
    }
    let collectionRequest = fetchCollection();

    const setSearchParams = () => {
        const obj = {
            includeExpansions: $libraryOptions.includeExpansions,
            useGeekRating: $libraryOptions.useGeekRating,
            selectedSort: $libraryOptions.selectedSort,
            sort: $libraryOptions.sort,
            ...$libraryOptions.filters
        }
        const url = Object.entries(obj)
            .filter(([key, value]) => {
                return !(typeof value === 'boolean' && !value);
            })
            .map(([key, value]) => {
                if (typeof value === 'boolean') {
                    return value
                        ? key
                        : ''
                } else {
                    return `${key}=${value}`;
                }
        }).join('&');

        window.history.replaceState(null, '', `?${url}`);
    }
</script>

<svelte:head>
    <title>{ displayName } Collection - Purple Meeple</title>
</svelte:head>
{#await collectionRequest}
    <div class="h-full flex flex-col justify-center items-center p-8">
        <lottie-player
            src="https://assets9.lottiefiles.com/packages/lf20_Wy80jjKz4n.json"
            background="transparent"
            speed="1"
            style="width: 300px; height: 300px;"
            loop
            autoplay
        ></lottie-player>
        <div class="text-center max-w-[750px] mt-8" transition:fade>
            {#if loadingState === 'collection'}
                <h3 class="mb-4">Loading collection...</h3>
                <p>
                    If this is the first time loading your collection,
                    this may take some time as BoardGameGeek has to process your collection first.
                </p>
            {:else if loadingState === 'games'}
                <h3 class="mb-4">
                    Collection loaded ({ collectionLength } games).<br/>Loading games {currentChunkRange}...
                </h3>
                <ProgressBar value={gameLoadingProgress} max={100} meter="bg-surface-900-50-token transition-[width]" />
            {/if}
        </div>
        {#if collectionLoadAttempts >= 2}
            <aside class="alert variant-ghost-primary mt-6 max-w-[750px]">
                <div class="alert-message text-center">
                    <p>
                        BoardGameGeek is still processing the collection request. If this continues to take some time, you can
                        safely leave this page and the request will continue to process in the background. The next time you
                        return, the process should be complete.
                    </p>
                </div>
            </aside>
        {/if}
    </div>
{:then col}
    {#if month === 11 || month === 0}
        <a
            class="w-full inline-block bg-gradient-to-r variant-gradient-primary-secondary text-center py-1 text-sm sm:text-base cursor-pointer"
            href={`/year-in-review/${username}`}
        >
            Tracking your plays on BGG? View your yearly stats!
        </a>
    {/if}
    <div class="w-full flex justify-center">
        <img class="h-8 w-8 mt-5 sm:h-14 sm:w-14 sm:mt-4 mr-2 -rotate-12" alt="Purple Meeple logo" src={logo} />
        <h1 class="font-title text-center text-4xl sm:text-6xl font-bold my-4">Purple Meeple</h1>
    </div>
    <div transition:fade class="flex flex-col items-center px-4 m-auto sm:max-w-[1020px]">
        <p class="unstyled text-center text-base sm:text-lg">
            {displayName} Collection
        </p>
        <p class="unstyled text-center text-sm sm:text-base">
            {$Library.gameCount} games, {$Library.expansionCount} expansions
        </p>
        <p class="unstyled text-center text-xs sm:text-sm">
            Showing {collection.length} of {$Library.data.length} items
        </p>
        {#if $Library.updateRequired}
            <p class="unstyled update-required text-center text-xs sm:text-sm">
                Changes detected, attempting to update...
            </p>
        {/if}
        <div class="w-full flex justify-end mb-4 mt-2 h-[42px]">
            <button class="btn variant-filled-secondary mr-4" on:click={openRandomGame}>
                <img src={Dice} alt="dice icon" class="h-6 w-6" />
            </button>
            <button class="btn variant-filled-secondary mr-4" on:click={openQR}>
                <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-6 h-6">
                    <path stroke-linecap="round" stroke-linejoin="round" d="M3.75 4.875c0-.621.504-1.125 1.125-1.125h4.5c.621 0 1.125.504 1.125 1.125v4.5c0 .621-.504 1.125-1.125 1.125h-4.5A1.125 1.125 0 013.75 9.375v-4.5zM3.75 14.625c0-.621.504-1.125 1.125-1.125h4.5c.621 0 1.125.504 1.125 1.125v4.5c0 .621-.504 1.125-1.125 1.125h-4.5a1.125 1.125 0 01-1.125-1.125v-4.5zM13.5 4.875c0-.621.504-1.125 1.125-1.125h4.5c.621 0 1.125.504 1.125 1.125v4.5c0 .621-.504 1.125-1.125 1.125h-4.5A1.125 1.125 0 0113.5 9.375v-4.5z" />
                    <path stroke-linecap="round" stroke-linejoin="round" d="M6.75 6.75h.75v.75h-.75v-.75zM6.75 16.5h.75v.75h-.75v-.75zM16.5 6.75h.75v.75h-.75v-.75zM13.5 13.5h.75v.75h-.75v-.75zM13.5 19.5h.75v.75h-.75v-.75zM19.5 13.5h.75v.75h-.75v-.75zM19.5 19.5h.75v.75h-.75v-.75zM16.5 16.5h.75v.75h-.75v-.75z" />
                </svg>
            </button>
            <button class="btn variant-filled-secondary mr-4" on:click={openOptions}>
                <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-6 h-6">
                    <path stroke-linecap="round" stroke-linejoin="round" d="M6 13.5V3.75m0 9.75a1.5 1.5 0 010 3m0-3a1.5 1.5 0 000 3m0 3.75V16.5m12-3V3.75m0 9.75a1.5 1.5 0 010 3m0-3a1.5 1.5 0 000 3m0 3.75V16.5m-6-9V3.75m0 3.75a1.5 1.5 0 010 3m0-3a1.5 1.5 0 000 3m0 9.75V10.5" />
                </svg>
            </button>
            <RadioGroup active="variant-filled-secondary" hover="hover:variant-soft-secondary">
                <RadioItem bind:group={displayType} name="display-type" value="list">
                    <ViewListIcon class="h-6 w-5" />
                </RadioItem>
                <RadioItem bind:group={displayType} name="display-type" value="grid">
                    <ViewGridIcon class="h-6 w-5" />
                </RadioItem>
            </RadioGroup>
        </div>
        {#if displayType === 'grid'}
            <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4 pb-4">
                {#each collection as game}
                    <GameCard {game} />
                {:else}
                    <p class="col-span-1 md:col-span-2 lg:col-span-3 text-center text-xl py-12">No results based on current filters</p>
                {/each}
            </div>
        {:else}
            {#each collection as game}
                <GameRow {game} />
            {:else}
                <p class="text-center text-xl py-12">No results based on current filters</p>
            {/each}
        {/if}
    </div>
{:catch error}
    <div class="h-full flex flex-col justify-center items-center">
        {#if error.status === 429}
            <aside class="alert variant-ghost-primary mt-6">
                <div class="alert-message text-center">
                    <p>Unable to load library for user <span class="font-semibold">{ username }</span></p>
                    <p>BGG has received a lot of requests and is busy. Please try again in a couple minutes</p>
                </div>
            </aside>
        {:else}
            <aside class="alert variant-ghost-primary mt-6">
                <div class="alert-message text-center">
                    <p>Unable to load library for user <span class="font-semibold">{ username }</span></p>
                    <p>{error.message || 'An unknown error has occurred.'}</p>
                </div>
            </aside>
            <button class="btn variant-filled-secondary mt-4" on:click={initFetchCollection}>Retry</button>
        {/if}
    </div>
{/await}

<style>
@-webkit-keyframes pulse {
    0% { opacity: 1; }
    70% { opacity: 1; }
    85% { opacity: 0.2; }
    100% { opacity: 1; }
}

@keyframes pulse {
    0% { opacity: 1; }
    70% { opacity: 1; }
    85% { opacity: 0.2; }
    100% { opacity: 1; }
}

.update-required {
    -webkit-animation: pulse 8s infinite ease-in-out;
    -o-animation: pulse 8s infinite ease-in-out;
    -ms-animation: pulse 8s infinite ease-in-out;
    -moz-animation: pulse 8s infinite ease-in-out;
    animation: pulse 8s infinite ease-in-out;
}
</style>