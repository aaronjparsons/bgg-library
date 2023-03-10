<script lang="ts">
    import type { ModalSettings, ModalComponent, ToastSettings } from '@skeletonlabs/skeleton';
    import { RadioGroup, RadioItem, modalStore, toastStore } from '@skeletonlabs/skeleton';
    import { ViewListIcon, ViewGridIcon } from '@rgossiaux/svelte-heroicons/solid';
    import { onDestroy } from 'svelte';
    import { fade } from 'svelte/transition';
    import { writable, type Writable } from 'svelte/store';
    import { page } from '$app/stores';
    import { goto } from '$app/navigation';
    import Dice from '$lib/icons/dice.png';
    import GameCard from "$lib/components/GameCard.svelte";
    import GameRow from "$lib/components/GameRow.svelte";
    import OptionsModal from "$lib/components/OptionsModal.svelte";
    import QRModal from "$lib/components/QRModal.svelte";
    import RandomGameModal from "$lib/components/RandomGameModal.svelte";
    import { getValue, sleep, getGameName } from "$lib/utils";
    import { Library, libraryOptions, ratingKey } from "$lib/store";

    const displayType: Writable<string> = writable('grid');
    const username = $page.params.username;
    let collection: Game[] = [];
    let collectionLoadAttempts = 0;
    let collectionLength = 0;
    let loadingState: string|null = null;

    const filterExpansions = (games: Game[]) => {
        const copy = [...games];
        return copy.filter(game => {
            if (!$libraryOptions.includeExpansions) {
                return game['@_type'] === 'boardgame';
            }
            return true;
        })
    }

    const setLibraryOptions = () => {
        const searchParams = new URLSearchParams(window.location.search);
        const filters = ['playtime', 'playerCount', 'rating', 'weight'];
        searchParams.forEach((value: string, key: string) => {
            if (filters.includes(key)) {
                $libraryOptions.filters[key] = value;
            } else {
                $libraryOptions[key] = key === 'includeExpansions' || key === 'useGeekRating'
                    ? value === 'true' : value;
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
        // Set options based on search params
        setLibraryOptions();

        collectionLoadAttempts = 0;
        loadingState = 'collection';
        let requestingCollection = true;
        let attempts = 0;
        let gameIds = [];
        while (requestingCollection) {
            const response = await fetch(`/api/collection?username=${username}`);

            if (response.ok) {
                if (response.status === 202) {
                    // BGG preparing
                    await sleep(10000);
                    collectionLoadAttempts++;
                } else {
                    const res = await response.json();
                    gameIds = res.gameIds;
                    requestingCollection = false;
                }
            } else {
                if (response.status === 404) {
                    // User doesn't exist
                    const toast: ToastSettings = {
                        message: 'Unable to find a user with that username. Please try again',
                        preset: 'secondary',
                        autohide: true,
                        timeout: 5000
                    };
                    toastStore.trigger(toast);
                    goto('/');
                    return;
                }

                return Promise.reject(response)
            }
        }

        collectionLength = gameIds.length;
        loadingState = 'games';
        const response = await fetch('/api/games', {
            method: 'POST',
            body: JSON.stringify({ gameIds }),
            headers: {
                'content-type': 'application/json'
            }
        });

        loadingState = null;
        if (response.ok) {
            const res = await response.json();
            $Library = {
                data: res,
                username,
                loaded: true
            };
            collection = sortAndFilter(res);
            setSearchParams();
            await sleep(150);
            console.log($Library.data[0])
            return res;
        } else {
            return Promise.reject(response)
        }
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
        const url = Object.entries(obj).map(([key, value]) => {
            return `${key}=${value}`;
        }).join('&');

        window.history.replaceState(null, '', `?${url}`);
    }
</script>

{#await collectionRequest}
    <div class="h-full flex flex-col justify-center items-center">
        <lottie-player
            src="https://assets9.lottiefiles.com/packages/lf20_Wy80jjKz4n.json"
            background="transparent"
            speed="1"
            style="width: 300px; height: 300px;"
            loop
            autoplay
        ></lottie-player>
        <div class="text-center max-w-[750px] mt-8">
            <h3 class="mb-4">
                {#if loadingState === 'collection'}
                    Loading collection...
                {:else if loadingState === 'games'}
                    Collection loaded. Loading data for { collectionLength } games...
                {/if}
            </h3>
            <p>
                If this is the first time loading your collection or if your collection has changed,
                this may take some time as BoardGameGeek has to process your collection first.
            </p>
        </div>
        {#if collectionLoadAttempts >= 2}
            <aside class="alert mt-6 max-w-[750px]">
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
    <h1 class="text-center text-4xl sm:text-7xl font-bold my-4">BGG-Library</h1>
    <div transition:fade class="flex flex-col items-center px-4 m-auto sm:max-w-[1020px]">
        <p class="unstyled text-center text-sm sm:text-base mb-2">
            Showing {collection.length} of {$Library.data.length} games
        </p>
        <div class="w-full flex justify-end mb-4 h-[42px]">
            <button class="btn btn-base btn-filled-secondary mr-4" on:click={openRandomGame}>
                <img src={Dice} alt="dice icon" class="h-6 w-6" />
            </button>
            <button class="btn btn-base btn-filled-secondary mr-4" on:click={openQR}>
                <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-6 h-6">
                    <path stroke-linecap="round" stroke-linejoin="round" d="M3.75 4.875c0-.621.504-1.125 1.125-1.125h4.5c.621 0 1.125.504 1.125 1.125v4.5c0 .621-.504 1.125-1.125 1.125h-4.5A1.125 1.125 0 013.75 9.375v-4.5zM3.75 14.625c0-.621.504-1.125 1.125-1.125h4.5c.621 0 1.125.504 1.125 1.125v4.5c0 .621-.504 1.125-1.125 1.125h-4.5a1.125 1.125 0 01-1.125-1.125v-4.5zM13.5 4.875c0-.621.504-1.125 1.125-1.125h4.5c.621 0 1.125.504 1.125 1.125v4.5c0 .621-.504 1.125-1.125 1.125h-4.5A1.125 1.125 0 0113.5 9.375v-4.5z" />
                    <path stroke-linecap="round" stroke-linejoin="round" d="M6.75 6.75h.75v.75h-.75v-.75zM6.75 16.5h.75v.75h-.75v-.75zM16.5 6.75h.75v.75h-.75v-.75zM13.5 13.5h.75v.75h-.75v-.75zM13.5 19.5h.75v.75h-.75v-.75zM19.5 13.5h.75v.75h-.75v-.75zM19.5 19.5h.75v.75h-.75v-.75zM16.5 16.5h.75v.75h-.75v-.75z" />
                </svg>
            </button>
            <button class="btn btn-base btn-filled-secondary mr-4" on:click={openOptions}>
                <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-6 h-6">
                    <path stroke-linecap="round" stroke-linejoin="round" d="M6 13.5V3.75m0 9.75a1.5 1.5 0 010 3m0-3a1.5 1.5 0 000 3m0 3.75V16.5m12-3V3.75m0 9.75a1.5 1.5 0 010 3m0-3a1.5 1.5 0 000 3m0 3.75V16.5m-6-9V3.75m0 3.75a1.5 1.5 0 010 3m0-3a1.5 1.5 0 000 3m0 9.75V10.5" />
                </svg>
            </button>
            <RadioGroup selected={displayType}>
                <RadioItem value="list">
                    <ViewListIcon class="h-5 w-5" />
                </RadioItem>
                <RadioItem value="grid">
                    <ViewGridIcon class="h-5 w-5" />
                </RadioItem>
            </RadioGroup>
        </div>
        {#if $displayType === 'grid'}
            <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
                {#each collection as game}
                    <GameCard {game} />
                {/each}
            </div>
        {:else}
            {#each collection as game}
                <GameRow {game} />
            {/each}
        {/if}
    </div>
{:catch error}
    <div class="h-full flex flex-col justify-center items-center">
        {#if error.status === 429}
            <aside class="alert mt-6">
                <div class="alert-message text-center">
                    <p>Unable to load library for user <span class="font-semibold">{ username }</span></p>
                    <p>BGG has received a lot of requests and is busy. Please try again in a couple minutes</p>
                </div>
            </aside>
        {:else}
            <aside class="alert mt-6">
                <div class="alert-message text-center">
                    <p>Unable to load library for user <span class="font-semibold">{ username }</span></p>
                    <p>An unknown error has occurred.</p>
                </div>
            </aside>
            <button class="btn btn-filled-secondary mt-4" on:click={initFetchCollection}>Retry</button>
        {/if}
    </div>
{/await}