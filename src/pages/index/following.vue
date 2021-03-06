<template>
  <ContentList
    :is-loading="isLoading"
    :items="items"
    :header-label="$t('FollowingPage.headerLabel')"
  >
    <template #empty>
      <div class="pt-24 px-40">
        <EmptyIcon class="block mx-auto" style="width:5.5rem" />
        <h1>{{ $t('FollowingPage.empty.title') }}</h1>
        <p>{{ $t('FollowingPage.empty.content') }}</p>
      </div>

      <div class="flex justify-center py-24">
        <NuxtLink
          class="btn btn--plain btn--auto-size btn--with-icon btn--icon-top flex-1 m-0 pt-32"
          :to="{ name: 'index' }"
        >
          <FeaturedIcon class="btn__icon w-24 p-24" />
          {{ $t('FollowingPage.empty.goToFeaturing') }}
        </NuxtLink>
      </div>
    </template>
  </ContentList>
</template>

<script>
import { mapActions, mapGetters } from 'vuex';

import ContentList from '~/components/ContentList';

import EmptyIcon from '~/assets/images/following-empty.svg';
import FeaturedIcon from '~/assets/icons/featured.svg';

export default {
  name: 'Following',
  scrollToTop: true,
  components: {
    ContentList,
    EmptyIcon,
    FeaturedIcon,
  },
  middleware: 'authenticated',
  data() {
    return {
      isLoading: true,
      isLoadingMore: false,
      isNoMoreUpdate: false,
      articles: [],
    };
  },
  computed: {
    ...mapGetters(['getFollowedAuthors', 'getFollowedArticles']),
    items() {
      return this.getFollowedArticles;
    },
  },
  mounted() {
    this.fetchContent();
    window.addEventListener('scroll', this.onScroll);
  },
  beforeDestroy() {
    window.removeEventListener('scroll', this.onScroll);
  },
  methods: {
    ...mapActions([
      'fetchReaderIndex',
      'fetchFollowedArticles',
      'updateFollowedArticles',
    ]),
    onScroll(e) {
      const { innerHeight: windowHeight, pageYOffset: scrollY } = window;
      const { scrollHeight } = document.documentElement;
      if (
        scrollY >= scrollHeight - windowHeight * 2 &&
        !this.isLoadingMore &&
        !this.isNoMoreUpdate
      ) {
        this.fetchMoreContent();
      }
    },
    async fetchContent() {
      try {
        this.isLoading = !this.getFollowedAuthors.length || !this.items.length;
        // const fetchReader = this.fetchReaderIndex();
        this.fetchReaderIndex();
        // if (!this.getFollowedAuthors.length) await fetchReader;
        const fetchArticles = this.fetchFollowedArticles();
        if (!this.articles.length) await fetchArticles;
      } catch (err) {
        console.error(err); // eslint-disable-line no-console
      } finally {
        this.isLoading = false;
      }
    },
    async fetchMoreContent() {
      try {
        this.isLoadingMore = true;
        const lastIndex = this.getFollowedArticles.length - 1;
        const lastTs = this.getFollowedArticles[lastIndex].ts;
        const list = await this.updateFollowedArticles({ before: lastTs });
        if (!list || !list.length) this.isNoMoreUpdate = true;
      } catch (err) {
        console.error(err); // eslint-disable-line no-console
      } finally {
        this.isLoadingMore = false;
      }
    },
  },
};
</script>
