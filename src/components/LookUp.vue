<script setup lang="ts">
import { ref } from "vue";
import {
    CompositeDidDocumentResolver,
    CompositeHandleResolver,
    DohJsonHandleResolver,
    PlcDidDocumentResolver,
    WebDidDocumentResolver,
    WellKnownHandleResolver,
} from "@atcute/identity-resolver";
import { AtpAgent } from "@atproto/api";

// handle resolution
const handleResolver = new CompositeHandleResolver({
    strategy: "race",
    methods: {
        dns: new DohJsonHandleResolver({
            dohUrl: "https://mozilla.cloudflare-dns.com/dns-query",
        }),
        http: new WellKnownHandleResolver(),
    },
});

const docResolver = new CompositeDidDocumentResolver({
    methods: {
        plc: new PlcDidDocumentResolver(),
        web: new WebDidDocumentResolver(),
    },
});

const userHandle = ref("");
const loading = ref(false);
const artists = ref<{ name: string; plays: number }[]>([]);
const tracks = ref<{ name: string; plays: number }[]>([]);
const totalSongs = ref(0);

const lookup = async () => {
    loading.value = true;
    try {
        const did = await handleResolver.resolve(
            userHandle.value as `${string}.${string}`,
        );

        if (did == undefined) {
            throw new Error("expected handle to resolve");
        }
        console.log(did); // did:plc:ewvi7nxzyoun6zhxrhs64oiz

        const doc = await docResolver.resolve(did);
        console.log(doc);

        // const handler = simpleFetchHandler({ service: });
        const agent = new AtpAgent({
            service: doc.service[0].serviceEndpoint as string,
        });
        let cursor = "";
        let plays = [];
        let response = await agent.com.atproto.repo.listRecords({
            repo: did,
            collection: "fm.teal.alpha.feed.play",
            limit: 100,
            cursor: cursor,
        });

        do {
            plays.push(...response.data.records);
            cursor = response.data.cursor;

            if (cursor) {
                response = await agent.com.atproto.repo.listRecords({
                    repo: did,
                    collection: "fm.teal.alpha.feed.play",
                    limit: 100,
                    cursor: cursor,
                });
            }
        } while (cursor);

        let inner_tracks = [];
        let inner_artists = [];
        for (const play of plays) {
            // spot-check if play is valid
            if (
                play.success == false ||
                play.value == undefined ||
                play.value.artistName ||
                play.value.trackName == undefined
            ) {
                continue;
            }
            // new version
            console.log(play);
            if (play.value?.artists) {
                for (const artist of play.value?.artists) {
                    let alreadyPlayed = inner_artists.find(
                        (a) => a.name === artist,
                    );
                    if (!alreadyPlayed) {
                        inner_artists.push({
                            name: artist.artistName,
                            plays: 1,
                        });
                    } else {
                        alreadyPlayed.plays++;
                    }
                }
            } else if (play.value?.artistNames) {
                // old version
                for (const arist of play.value?.artistNames) {
                    let alreadyPlayed = inner_artists.find(
                        (a) => a.name === arist,
                    );
                    if (!alreadyPlayed) {
                        inner_artists.push({ name: arist, plays: 1 });
                    } else {
                        alreadyPlayed.plays++;
                    }
                }
            }

            let alreadyPlayed = inner_tracks.find(
                (a) => a.name === play.value.trackName,
            );
            if (!alreadyPlayed && play?.value) {
                inner_tracks.push({
                    name: play.value.trackName,
                    artist: play.value?.artists
                        ? play.value.artists[0].artistName
                        : play.value.artistNames[0],
                    plays: 1,
                });
            } else {
                alreadyPlayed.plays++;
            }
        }

        artists.value = inner_artists
            .sort((a, b) => b.plays - a.plays)
            .slice(0, 10);
        tracks.value = inner_tracks
            .sort((a, b) => b.plays - a.plays)
            .slice(0, 10);
        totalSongs.value = plays.length;
    } finally {
        loading.value = false;
    }
};
</script>

<template>
    <div class="container mx-auto p-4 text-center">
        <h1
            class="text-5xl font-bold mb-2 bg-gradient-to-r from-teal-400 to-teal-600 text-transparent bg-clip-text"
        >
            Teal Wrapped
        </h1>
        <p class="text-sm text-gray-500 mb-8">
            Mostly not affiliated with teal.fm™
        </p>
        <div class="join w-full justify-center">
            <input
                v-model="userHandle"
                type="text"
                placeholder="alice.bsky.social"
                class="input input-bordered join-item w-1/2 max-w-xs"
            />
            <button
                @click="lookup"
                class="btn join-item bg-teal-500 hover:bg-teal-600 text-white"
            >
                That's a wrap
            </button>
        </div>
        <div class="w-full justify-center">
            <span
                v-if="loading"
                class="loading loading-dots loading-lg mt-8"
            ></span>
            <div v-if="tracks.length > 0" class="mt-8">
                <h2 class="text-2xl font-bold mb-4">
                    Top Songs out of {{ totalSongs }}
                </h2>
                <div class="overflow-x-auto">
                    <table class="table w-full">
                        <thead>
                            <tr>
                                <th>Plays</th>
                                <th>Song</th>
                            </tr>
                        </thead>
                        <tbody>
                            <tr v-for="track in tracks" :key="track.name">
                                <td>{{ track.plays }}</td>
                                <td>{{ track.name }} by {{ track.artist }}</td>
                            </tr>
                        </tbody>
                    </table>
                </div>
            </div>
            <div v-if="artists.length > 0" class="mt-8">
                <h2 class="text-2xl font-bold mb-4">Top Artists</h2>
                <div class="overflow-x-auto">
                    <table class="table w-full">
                        <thead>
                            <tr>
                                <th>Plays</th>
                                <th>Artist</th>
                            </tr>
                        </thead>
                        <tbody>
                            <tr v-for="artist in artists" :key="artist.name">
                                <td>{{ artist.plays }}</td>
                                <td>{{ artist.name }}</td>
                            </tr>
                        </tbody>
                    </table>
                </div>
            </div>
        </div>
    </div>
</template>

<style scoped>
.container {
    min-height: 50vh;
    display: flex;
    flex-direction: column;
    justify-content: center;
}
</style>
