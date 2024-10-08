<template>
  <div id="page-wrapper" class="bg-bg page overflow-y-auto p-4 md:p-8" :class="streamLibraryItem ? 'streaming' : ''">
    <div class="max-w-6xl mx-auto">
      <div class="flex flex-wrap sm:flex-nowrap justify-center mb-6">
        <div class="w-48 min-w-48">
          <div class="w-full h-60">
            <covers-author-image :author="author" rounded="0" />
          </div>
        </div>
        <div class="flex-grow py-4 sm:py-0 px-4 md:px-8">
          <div class="flex items-center mb-8">
            <h1 class="text-2xl">{{ author.name }}</h1>

            <button v-if="userCanUpdate" class="w-8 h-8 rounded-full flex items-center justify-center mx-4 cursor-pointer text-gray-300 hover:text-warning transform hover:scale-125 duration-100" @click="editAuthor">
              <span class="material-symbols text-base">edit</span>
            </button>
          </div>

          <!-- Alias or Original Author Section -->
          <div v-if="author.is_alias_of !== null && author.is_alias_of != 0" class="mb-4">
            <p class="text-white text-opacity-60 uppercase text-xs mb-2">Origin Author</p>
            <nuxt-link :to="`/author/${author.is_alias_of}`" class="alias-box inline-block p-2 rounded mb-2 text-white">
              {{ author.originalAuthor.name }}
            </nuxt-link>
          </div>

          <div v-else-if="!author.is_alias_of && author.aliases.length" class="mb-4">
            <p class="text-white text-opacity-60 uppercase text-xs mb-2">Aliases</p>
            <div v-for="alias in author.aliases" :key="alias.id">
              <nuxt-link :to="`/author/${alias.id}`">
                <div class="inline-block alias-box p-2 rounded mb-2 text-white">
                  {{ alias.name }}
                </div>
              </nuxt-link>
            </div>
          </div>
          <div v-else-if="author.is_alias_of === 0" class="mb-4">
            <p class="text-white text-opacity-60 text-xs mb-2">Author {{ author.name }} is a combined auther</p>
            <p class="text-white text-opacity-60 text-xs mb-2">The following authors use this alias write these books.</p>
            <div v-for="originalAuthor in author.combinedOriginalAuthor" :key="originalAuthor.id">
              <nuxt-link :to="`/author/${originalAuthor.id}`">
                <div class="inline-block alias-box p-2 rounded mb-2 text-white">
                  {{ originalAuthor.name }}
                </div>
              </nuxt-link>
            </div>
          </div>
          <div v-if="author.combinedAlias.length !== 0" class="mb-4">
            <p class="text-white text-opacity-60 text-xs mb-2">Author {{ author.name }} has following combined alias</p>
            <div v-for="combinedAlias in author.combinedAlias" :key="combinedAlias.id">
              <nuxt-link :to="`/author/${combinedAlias.id}`">
                <div class="inline-block alias-box p-2 rounded mb-2 text-white">
                  {{ combinedAlias.name }}
                </div>
              </nuxt-link>
            </div>
          </div>

          <p v-if="author.description" class="text-white text-opacity-60 uppercase text-xs mb-2">{{ $strings.LabelDescription }}</p>
          <p ref="description" id="author-description" class="text-white max-w-3xl text-base whitespace-pre-wrap" :class="{ 'show-full': showFullDescription }">{{ author.description }}</p>
          <button v-if="isDescriptionClamped" class="py-0.5 flex items-center text-slate-300 hover:text-white" @click="showFullDescription = !showFullDescription">
            {{ showFullDescription ? $strings.ButtonReadLess : $strings.ButtonReadMore }} <span class="material-symbols text-xl pl-1">{{ showFullDescription ? 'expand_less' : 'expand_more' }}</span>
          </button>
        </div>
      </div>

      <div class="py-4">
        <widgets-item-slider :items="libraryItems" shelf-id="author-books" :bookshelf-view="$constants.BookshelfView.AUTHOR">
          <nuxt-link :to="`/library/${currentLibraryId}/bookshelf?filter=authors.${$encode(author.id)}`" class="hover:underline">
            <h2 class="text-lg">{{ libraryItems.length }} {{ $strings.LabelBooks }}</h2>
          </nuxt-link>
        </widgets-item-slider>
      </div>

      <div v-for="series in authorSeries" :key="series.id" class="py-4">
        <widgets-item-slider :items="series.items" :shelf-id="series.id" :bookshelf-view="$constants.BookshelfView.AUTHOR">
          <nuxt-link :to="`/library/${currentLibraryId}/series/${series.id}`" class="hover:underline">
            <h2 class="text-lg">{{ series.name }}</h2>
          </nuxt-link>
          <p class="text-white text-opacity-40 text-base px-2">{{ $strings.LabelSeries }}</p>
        </widgets-item-slider>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  async asyncData({ store, app, params, redirect, query }) {
    const author = await app.$axios.$get(`/api/authors/${params.id}?include=items,series`).catch((error) => {
      console.error('Failed to get author', error)
      return null
    })
    let originalAuthor = null
    if (author.is_alias_of != null && author.is_alias_of != 0) {
      originalAuthor = await app.$axios.$get(`/api/authors/${author.is_alias_of}?include=items,series`).catch((error) => {
        console.error('Failed to get original author', error)
        return null
      })
    }

    const aliases = await app.$axios.$get(`/api/authors/${params.id}/alias`).catch((error) => {
      console.error('Failed to get aliases', error)
      return null
    })
    const combinedOriginalAuthor = await app.$axios.$get(`/api/authors/${params.id}/origins`).catch((error) => {
      console.error('Failed to get combined original author', error)
      return null
    })

    const combinedAlias = await app.$axios.$get(`/api/authors/${params.id}/combined_alias`).catch((error) => {
      console.error('Failed to get combined alias', error)
      return null
    })
    //console.log('combinedAlias----------------------', combinedAlias)

    if (!author) {
      return redirect(`/library/${store.state.libraries.currentLibraryId}/authors`)
    }

    if (query.library) {
      store.commit('libraries/setCurrentLibrary', query.library)
    }

    return {
      author: {
        ...author,
        originalAuthor: originalAuthor || null,
        aliases: aliases || [],
        combinedOriginalAuthor,
        combinedAlias: combinedAlias
      }
    }
  },
  data() {
    return {
      isDescriptionClamped: false,
      showFullDescription: false
    }
  },
  computed: {
    streamLibraryItem() {
      return this.$store.state.streamLibraryItem
    },
    currentLibraryId() {
      return this.$store.state.libraries.currentLibraryId
    },
    libraryItems() {
      return this.author.libraryItems || []
    },
    authorSeries() {
      return this.author.series || []
    },
    userCanUpdate() {
      return this.$store.getters['user/getUserCanUpdate']
    }
  },
  methods: {
    checkDescriptionClamped() {
      if (!this.$refs.description) return
      this.isDescriptionClamped = this.$refs.description.scrollHeight > this.$refs.description.clientHeight
    },
    editAuthor() {
      this.$store.commit('globals/showEditAuthorModal', this.author)
    },
    authorUpdated(author) {
      if (author.id === this.author.id) {
        console.log('Author was updated', author)
        this.author = {
          ...author,
          series: this.authorSeries,
          libraryItems: this.libraryItems
        }
        this.$nextTick(this.checkDescriptionClamped)
      }
    },
    authorRemoved(author) {
      if (author.id === this.author.id) {
        console.warn('Author was removed')
        this.$router.replace(`/library/${this.currentLibraryId}/authors`)
      }
    }
  },
  mounted() {
    if (!this.author) this.$router.replace('/')
    this.checkDescriptionClamped()

    this.$root.socket.on('author_updated', this.authorUpdated)
    this.$root.socket.on('author_removed', this.authorRemoved)
  },
  beforeDestroy() {
    this.$root.socket.off('author_updated', this.authorUpdated)
    this.$root.socket.off('author_removed', this.authorRemoved)
  }
}
</script>


<style scoped>
#author-description {
  overflow: hidden;
  display: -webkit-box;
  -webkit-box-orient: vertical;
  -webkit-line-clamp: 4;
  max-height: 6.25rem;
  transition: all 0.3s ease-in-out;
}
#author-description.show-full {
  -webkit-line-clamp: unset;
  max-height: 999rem;
}

.alias-box {
  background-color: rgba(27, 27, 27, 0.6);
  color: #fff;
  padding: 0.5rem 1rem;
  border-radius: 0.25rem;
  margin: 0.25rem;
}
</style>
