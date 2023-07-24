<template>
  <div class="prose mx-auto p-8">
    <Head>
      <Title>Your Favorite Songs - Web5</Title>
    </Head>
    <h2 class="text-2xl font-bold mb-4 text-center">Favorite Songs</h2>
    <div class="bg-white rounded-xl shadow-lg p-8">
      <!-- Show loading screen -->
      <div class="text-center mt-4" v-if="isLoading">
        <span class="text-md mb-4">Fetching favorite songs...</span>
      </div>
      <!-- Empty favorite songs list? Show form to add 3 fav songs  -->
      <div class="text-center mt-4" v-else-if="songs.length === 0 && !isLoading">
        <span class="text-xl font-bold mb-4">No songs added yet!</span>
        <p class="text-sm">Please use the form below to add your favorite songs and artists.</p>
        <div class="col-span-2">
          <h4 class="text-md font-bold mb-4">Add Your Top 3 Favorite Songs and Artists</h4>
          <form @submit.prevent="addSongs">
            <div v-for="index in 3" :key="index">
              <h3 class="text-xl font-bold mb-2">Song {{ index }}</h3>
              <div class="grid grid-cols-2 gap-4">
                <div>
                  <label class="block text-sm font-medium text-gray-700">Song Title</label>
                  <input v-model="newSongs[index - 1].title" type="text" class="mt-1 p-2 block w-full rounded-md border border-gray-300">
                </div>
                <div>
                  <label class="block text-sm font-medium text-gray-700">Artist</label>
                  <input v-model="newSongs[index - 1].artist" type="text" class="mt-1 p-2 block w-full rounded-md border border-gray-300">
                </div>
              </div>
            </div>
            <div class="mt-4">
              <button type="submit" class="px-4 py-2 bg-blue-500 text-white rounded-md">Add Songs</button>
            </div>
          </form>
        </div>
      </div>
      <!-- See your fav songs and add a new one -->
      <div v-else>
        <div class="col-span-2 mb-4">
          <h4 class="text-md font-bold mb-4">Add New Song and Artist</h4>
          <form @submit.prevent="addSong">
            <div class="grid grid-cols-2 gap-4">
              <div>
                <label class="block text-sm font-medium text-gray-700">Song Title</label>
                <input v-model="newSong.title" type="text" class="mt-1 p-2 block w-full rounded-md border border-gray-300">
              </div>
              <div>
                <label class="block text-sm font-medium text-gray-700">Artist</label>
                <input v-model="newSong.artist" type="text" class="mt-1 p-2 block w-full rounded-md border border-gray-300">
              </div>
            </div>
            <div class="mt-4">
              <button type="submit" class="px-4 py-2 bg-blue-500 text-white rounded-md w-full">Add Song</button>
            </div>
          </form>
        </div>
        <hr />
        <div v-for="(song, index) in songs" :key="index" class="flex items-center not-prose mb-4 justify-between">
          <div class="flex space-x-4">
            <img :src="song.data.image || 'https://via.placeholder.com/150'" alt="Cover Art" class="h-16 w-16">
            <div>
              <p class="font-bold">{{ song.data.name }}</p>
              <p class="text-sm text-gray-500">{{ song.data.byArtist.name }}</p>
            </div>
          </div>
          <div>
            <button @click="deleteSong(song)" class="px-4 py-2 bg-red-500 text-white rounded-md text-sm float-right">Delete</button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
<script setup>
import { Web5 }from '@tbd54566975/web5';
const runtimeConfig = useRuntimeConfig();

let web5;
let myDid;

const songs = ref([]);
const newSong = ref({
  title: "",
  artist: "",
});

const newSongs = ref([
  { title: "", artist: "" },
  { title: "", artist: "" },
  { title: "", artist: "" },
])

// set up loading view
const isLoading = ref(true)

onBeforeMount(async () => {
  ({ web5, did: myDid } = await Web5.connect());

  const { records } = await web5.dwn.records.query({
    message: {
      filter: {
        schema: 'https://schema.org/MusicRecording'
      },
      dateSort: 'createdAscending'
    }
  });

  // Add entry to Songs array
  for (let record of records) {
    const data = await record.data.json();
    const song = { record, data, id: record.id };
    songs.value.push(song);
  }

  isLoading.value = false;
});

const getCoverArt = async (title, artist) => {
  // fetch cover art from LastFM API
  const trackinfo = await $fetch('https://ws.audioscrobbler.com/2.0/', {
    method: 'GET',
    params: { 
      'api_key': runtimeConfig.public.LAST_FM_API_KEY,
      'method': 'track.getInfo',
      'format': 'json',
      'track': title.trim(),
      'artist': artist.trim()
    }
  });

  return trackinfo.track?.album?.image[2]["#text"] || false;
}

const addSong = async () => {
  let songData = {};

 if (newSong.value.title.trim() && newSong.value.artist.trim()) {
    const title = newSong.value.title.trim();
    const artist = newSong.value.artist.trim();

    const coverArtUrl = await getCoverArt(title, artist);

    songData = {
      name: title,
      byArtist: {
        name: artist
      },
      image: coverArtUrl || "https://via.placeholder.com/150",
    };

    storeSong(songData)
    newSong.value.title = "";
    newSong.value.artist = "";
  }
  else {
    alert("Please enter a favorite song AND artist.");
  }

}

const addSongs = async () => {
  // Add more than one song at a go.
  const validSongs = newSongs.value.filter(song => song.title.trim() && song.artist.trim());
  if (validSongs.length === 3) {

    for (const song of validSongs) {
      const title = song.title.trim();
      const artist = song.artist.trim();

      const coverArtUrl = await getCoverArt(title, artist);

      const songData = {
        name: title,
        byArtist: {
          name: artist
        },
        image: coverArtUrl || "https://via.placeholder.com/150",
      };
      storeSong(songData);
    };

    newSongs.value = [
      { title: "", artist: "" },
      { title: "", artist: "" },
      { title: "", artist: "" },
    ];
  } else {
    alert("Please enter all three favorite songs and artists.");
  }
}

const storeSong = async (songData) => {
  // Create the record in DWN
  const { record } = await web5.dwn.records.create({
    data    : songData,
    message : {
      schema     : 'https://schema.org/MusicRecording',
      dataFormat : 'application/json'
    }
  });


  // add record ID so we can use to delete later
  const data = await record.data.json();
  const song = { record, data, id: record.id };
  songs.value.push(song);

  return;
}

const deleteSong = async (songItem) => {
  let deletedSong;
  let index = 0;

  for (let song of songs.value) {
    if (songItem.id === song.id) {
      deletedSong = song;
      break;
    }
    index ++;
  }

  songs.value.splice(index, 1);

  // Delete the record in DWN
  await web5.dwn.records.delete({
    message: {
      recordId: deletedSong.id
    }
  });
}
</script>