<template>
  <div class="dev-portal-services">
    <div
      class="services-top-section flex flex-col items-center justify-center py-16 bg-section_colors-hero"
      :style="catalog_cover_style"
    >
      <h4 class="services-welcome mb-4 font-normal color-text_colors-secondary text-2xl">
        {{ welcome_message }}
      </h4>
      <h1 class="services-title mb-5 font-normal color-text_colors-hero text-4xl">
        {{ primary_header }}
      </h1>
      <div class="w-full max-w-lg mx-auto inline-flex">
        <form
          id="searchServicesForm"
          @submit.prevent="searchServices"
          @reset.prevent="searchServices"
        >
          <KInput
            v-model="searchString"
            class="k-input--full"
            size="small"
            type="search"
            :placeholder="helpText.search"
            data-testid="catalog-search"
            form="searchServicesForm"
            @input="searchServices"
          />
          <KButton
            form="searchServicesForm"
            appearance="primary"
            data-testid="catalog-search-button"
            type="submit"
            size="small"
            :disabled="loading"
            :is-rounded="false"
          >
            {{ searchString !== '' && loading ? helpText.searching : helpText.search }}
          </KButton>
        </form>
      </div>
    </div>
    <Catalog
      :services="services"
      :cards-per-page="cardsPerPage"
      :total-count="totalCount"
      :search-triggered="searchTriggered"
      :loading="loading"
      @active-view-changed="catalogViewChanged"
      @cards-page-changed="catalogPageChanged"
    />
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, onBeforeMount } from 'vue'
import usePortalApi from '@/hooks/usePortalApi'
import Catalog from '@/components/Catalog.vue'
import { sortBy, SortOrder, SortType } from '@/helpers/sortBy'
import { debounce } from '@/helpers/debounce'
import { useI18nStore, CustomProduct } from '@/stores'

export default defineComponent({
  name: 'Services',
  components: { Catalog },

  setup () {
    const catalog_cover_style = ref<{backgroundImage:string}>({ backgroundImage: '' })
    const welcome_message = ref('')
    const primary_header = ref('')
    const cardsPerPage = ref(12)
    const searchString = ref('')
    const services = ref<CustomProduct[]>([])
    const totalCount = ref<number>(undefined)
    const loading = ref<boolean>(null)
    const searchTriggered = ref<boolean>(false)
    const catalogView = ref<string>(undefined)
    const catalogPageNumber = ref(1)
    const helpText = useI18nStore().state.helpText.services

    const { portalApiV2 } = usePortalApi()

    const loadAppearance = async () => {
      return portalApiV2.value.service.portalApi.getPortalAppearance().then(res => {
        const portalVariables = res.data.variables.catalog

        if (portalVariables.welcome_message) {
          welcome_message.value = portalVariables.welcome_message.text
        }

        if (portalVariables.primary_header) {
          primary_header.value = portalVariables.primary_header.text
        }

        if (portalVariables.cover) {
          const imageUrl = portalApiV2.value.getApiLink('/api/v2/portal/catalog-cover')

          catalog_cover_style.value.backgroundImage = `url(${imageUrl})`
        }
      }).catch(e => { console.error('Failed to load appearance.', e) }).then(defaultHeaders)
    }

    const defaultHeaders = () => {
      if (!welcome_message.value) {
        welcome_message.value = 'Welcome to our API Portal!'
      }

      if (!primary_header.value) {
        primary_header.value = 'Start building and innovating with our APIs'
      }
    }

    const searchServices = debounce(async () => {
      searchTriggered.value = true
      catalogPageNumber.value = 1

      try {
        return await fetchServices()
      } finally {
        searchTriggered.value = false
      }
    })

    const fetchServices = async () => {
      loading.value = true

      try {
        try {
          const { data: portalEntities } = await portalApiV2.value.service.searchApi.searchPortalEntities({
            indices: 'product-catalog',
            q: searchString.value,
            pageNumber: catalogPageNumber.value,
            pageSize: cardsPerPage.value,
            join: 'versions'
          })
          const { data: sources, meta } = portalEntities

          services.value = sources.map(({ source }) => {
            const versions = [...source.versions]
              .sort(sortBy('created_at', SortOrder.ASC, SortType.DATE))
              .map(version => version.name)

            return {
              id: source.id,
              title: source.name,
              versions,
              description: source.description,
              hasDocumentation: source.has_documentation
            }
          })
          totalCount.value = meta.page.total
        } catch (e) {
          console.error('failed to find Service Packages', e)
        }
      } finally {
        loading.value = null
      }
    }

    const catalogViewChanged = (viewType: string) => {
      services.value = []
      catalogView.value = viewType
      fetchServices()
    }

    const catalogPageChanged = (pageNumber: number) => {
      catalogPageNumber.value = pageNumber
      if (!searchTriggered.value) {
        fetchServices()
      }
    }

    onBeforeMount(async () => {
      await loadAppearance()
    })

    return {
      catalog_cover_style,
      welcome_message,
      primary_header,
      cardsPerPage,
      searchString,
      services,
      totalCount,
      loading,
      searchTriggered,
      catalogView,
      catalogPageNumber,
      helpText,
      searchServices,
      catalogViewChanged,
      catalogPageChanged
    }
  }
})
</script>

<style lang="scss">
.dev-portal-services {
  form {
    display: flex;
    width: 100%;
  }
  .services-top-section {
    border-bottom: 1px solid var(--section_colors-stroke);

    .k-input {
      fill: var(--text_colors-accent);
      border-radius: 3px 0 0 3px !important;
      &::-webkit-input-placeholder { color:var(--text_colors-secondary) !important; }
      &::-moz-placeholder { color:var(--text_colors-secondary) !important; }
      &::-ms-placeholder { color:var(--text_colors-secondary) !important; }
      &::placeholder { color:var(--text_colors-secondary) !important; }
    }
    .k-button {
      border-radius: 0 3px 3px 0;
      font-weight: normal !important;
    }
  }
}
</style>
