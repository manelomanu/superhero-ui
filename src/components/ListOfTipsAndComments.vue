<template>
  <div>
    <div class="profile__actions">
      <a
        :class="{ active: activeTab === 'tips' }"
        @click="setActiveTab('tips')"
      >
        {{ $t('tips') }}
      </a>
      <a
        :class="{ active: activeTab === 'comments' }"
        @click="setActiveTab('comments')"
      >
        {{ $t('comments') }}
      </a>
    </div>
    <div class="position-relative">
      <div
        v-if="activeTab === 'tips'"
        class="tips__container"
      >
        <TipsPagination
          tip-sort-by="latest"
          :address="address"
        />
      </div>
      <div
        v-if="activeTab === 'comments'"
        class="tips__container"
      >
        <Loading
          v-if="showLoading"
          class="loading-position-absolute"
        />
        <div
          v-if="showNoResultsMsg"
          class="no-results text-center w-100 mt-3"
          :class="[error ? 'error' : '']"
        >
          {{ $t('views.UserProfileView.NoActivity') }}
        </div>
        <TipComment
          v-for="(comment, index) in comments"
          :key="index"
          :comment="comment"
          :sender-link="openExplorer(comment.author)"
        />
      </div>
    </div>
  </div>
</template>

<script>
import { mapState } from 'vuex';
import Backend from '../utils/backend';
import { EventBus } from '../utils/eventBus';
import { EXPLORER_URL } from '../config/constants';
import Loading from './Loading.vue';
import TipsPagination from './TipsPagination.vue';
import TipComment from './tipRecords/TipComment.vue';

export default {
  components: {
    TipsPagination,
    Loading,
    TipComment,
  },
  props: { address: { type: String, required: true } },
  data: () => ({
    explorerUrl: `${EXPLORER_URL}account/transactions/`,
    showLoading: false,
    error: false,
    comments: [],
    activeTab: 'tips',
  }),
  computed: {
    ...mapState(['loading']),
    showNoResultsMsg() {
      return this.activeTab === 'comments'
        && this.comments.length === 0 && !this.showLoading && !this.loading.tips;
    },
  },
  mounted() {
    this.reloadData();

    EventBus.$on('reloadData', () => {
      this.reloadData();
    });

    const interval = setInterval(() => this.reloadData(), 120 * 1000);
    this.$once('hook:beforeDestroy', () => clearInterval(interval));
  },
  methods: {
    setActiveTab(tab) {
      this.activeTab = tab;
    },
    openExplorer(address) {
      return this.explorerUrl + address;
    },
    reloadData() {
      this.showLoading = true;
      Backend.getAllComments()
        .then((allComments) => {
          this.showLoading = false;
          this.error = false;
          this.comments = allComments.filter(
            (comment) => comment.author === this.address,
          );
        })
        .catch((e) => {
          console.error(e);
          this.error = true;
          this.showLoading = false;
        });
    },
  },
};
</script>

<style lang="scss" scoped>
.profile__actions {
  background-color: $actions_ribbon_background_color;
  margin-top: 0.125rem;
  padding-left: 1rem;
  position: sticky;
  top: 3.1rem;
  z-index: 21;

  a {
    color: $light_font_color;
    display: inline-block;
    font-weight: 600;
    margin-right: 0.5rem;
    padding: 0.5rem;

    &:last-child {
      margin-right: 0;
    }

    &:hover {
      color: $primary_color;
      cursor: pointer;
    }

    &.active {
      border-bottom: 2px solid $custom_links_color;
      color: $custom_links_color;
    }
  }
}
</style>
