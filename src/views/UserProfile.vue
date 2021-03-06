<template>
  <Page
    :loading="loading.initial"
    back
  >
    <div class="profile__page">
      <div class="profile__section clearfix">
        <div class="profile__row">
          <div
            class="profile__header"
            :class="{ 'profile__editable': account === address }"
          >
            <div
              v-if="account === address"
              class="edit__buttons"
            >
              <button
                v-if="!editMode"
                class="profile__button edit__button"
                type="button"
                :title="$t('views.UserProfileView.EditProfile')"
                @click="editMode = true"
              >
                <img src="../assets/buttonEdit.svg">
              </button>
              <button
                v-if="editMode"
                type="button"
                class="profile__button cancel__button"
                :title="$t('cancel')"
                @click="resetEditedValues"
              >
                <img src="../assets/buttonCancel.svg">
              </button>
              <button
                v-if="editMode"
                type="button"
                class="profile__button save__button"
                :title="$t('views.UserProfileView.Save')"
                @click="saveProfile"
              >
                <img src="../assets/buttonSave.svg">
              </button>
            </div>
            <div class="profile__image">
              <div
                v-if="account === address"
                :title="address"
              >
                <Avatar :address="address" />
                <label
                  v-if="!editMode"
                  class="profile__button avatar__button"
                  :title="$t('views.UserProfileView.ChangeAvatar')"
                >
                  <img src="../assets/buttonPhoto.svg">
                  <input
                    id="file-input"
                    type="file"
                    name="avatar"
                    accept="image/png, image/jpeg"
                    @change="uploadAvatar($event)"
                  >
                </label>
                <button
                  v-if="!editMode"
                  class="profile__button delete_avatar__button"
                  :title="$t('views.UserProfileView.DeleteAvatar')"
                  @click="deleteAvatar"
                >
                  <img src="../assets/buttonCancel.svg">
                </button>
              </div>
              <div
                v-else
                :title="address"
              >
                <Avatar :address="address" />
                <TipInput
                  v-if="!editMode"
                  :user-address="address"
                  class="avatar__button profile__button"
                />
              </div>
            </div>
            <div
              class="profile__info"
            >
              <a
                class="profile__username"
                target="_blank"
                :href="openExplorer(address)"
                :title="address"
              >
                <div class="chain">
                  {{ userChainName ? userChainName : $t('FellowSuperhero') }}
                </div>
                <div>{{ address }}</div>
              </a>
              <div
                v-if="userStats && hasCreationDate"
                class="joined"
              >
                <span>{{ $t('views.UserProfileView.Joined') }}</span>
                <time :datetime="joinedAtISO">{{ joinedAt }}</time>
              </div>
            </div>
          </div>
          <div
            v-if="!editMode"
            class="profile__description"
          >
            {{ profile.biography }}
          </div>
          <div
            v-if="editMode"
            class="profile__description"
          >
            <textarea
              v-model="profile.biography"
              :maxlength="maxLength"
              :rows="profile.biography.split('\n').length || 1"
              :placeholder="$t('views.UserProfileView.EditBiography')"
            />
            <div
              class="text-length"
              :class="{ 'error': profile.biography.length > maxLength }"
            >
              {{ countLength }}
            </div>
          </div>
        </div>
        <div
          v-if="userStats"
          class="profile__stats"
        >
          <div
            v-for="(divClass, index) in ['tip_stats', 'stats']"
            :key="index"
            :class="divClass"
          >
            <div
              v-for="(stat, idx) in divClass === 'tip_stats' ? tipStats : showedStats"
              :key="idx"
              :class="divClass === 'tip_stats' ? 'tips_stats_block' : 'stat_block'"
            >
              <span
                v-if="stat.value || stat.image"
                class="stat-value"
              >
                {{ stat.value }}
              </span>
              <span class="stat-title">
                <img
                  v-if="stat.image"
                  :src="stat.image"
                >
                {{ stat.title }}
              </span>
              <AeAmountFiat
                v-if="stat.amount"
                :amount="stat.amount"
                class="stat-value"
              />
            </div>
          </div>
        </div>
      </div>
      <ListOfTipsAndComments :address="address" />
    </div>
  </Page>
</template>

<script>
import { mapState } from 'vuex';
import Backend from '../utils/backend';
import { createDeepLinkUrl } from '../utils/util';
import Page from '../components/layout/Page.vue';
import { client } from '../utils/aeternity';
import AeAmountFiat from '../components/AeAmountFiat.vue';
import { EXPLORER_URL } from '../config/constants';
import ListOfTipsAndComments from '../components/ListOfTipsAndComments.vue';
import Avatar from '../components/Avatar.vue';
import { EventBus } from '../utils/eventBus';
import TipInput from '../components/TipInput.vue';
import SuccessIcon from '../assets/verifiedUrl.svg';

export default {
  components: {
    ListOfTipsAndComments,
    AeAmountFiat,
    Page,
    Avatar,
    TipInput,
  },
  props: {
    address: { type: String, required: true },
  },
  data() {
    return {
      maxLength: 250,
      explorerUrl: `${EXPLORER_URL}account/transactions/`,
      error: false,
      userStats: null,
      editMode: false,
      userCommentCount: 0,
      profile: {
        biography: '',
        createdAt: '',
      },
    };
  },
  computed: {
    ...mapState(['useSdkWallet', 'account', 'chainNames', 'loading']),
    userChainName() {
      return this.chainNames[this.address];
    },
    hasCreationDate() {
      return this.profile.createdAt.length > 0;
    },
    joinedAtISO() {
      try {
        return new Date(this.profile.createdAt).toISOString();
      } catch (e) {
        return '';
      }
    },
    joinedAt() {
      try {
        return new Date(this.profile.createdAt).toLocaleDateString('en-US', {
          year: 'numeric',
          month: 'long',
          day: 'numeric',
        });
      } catch (e) {
        return '';
      }
    },
    countLength() {
      return `${this.profile.biography.length}/${this.maxLength}`;
    },
    tipStats() {
      return [
        {
          value: this.userStats.tipsLength + this.userStats.retipsLength,
          title: this.$t('views.UserProfileView.TipsSent'),
          amount: this.userStats.totalTipAmount,
        },
        {
          title: this.$t('views.UserProfileView.ClaimedAmount'),
          amount: this.userStats.claimedAmount,
        },
        {
          title: this.$t('views.UserProfileView.UnclaimedAmount'),
          amount: this.userStats.unclaimedAmount,
        },
      ];
    },
    showedStats() {
      return [
        { value: this.userStats.userComments, title: this.$t('views.UserProfileView.Comments') },
        { value: this.userStats.tipsLength, title: this.$t('views.UserProfileView.TipsReceived') },
        { value: this.userStats.retipsLength, title: this.$t('views.UserProfileView.RetipsSent') },
        {
          value: this.userStats.claimedUrlsLength,
          image: SuccessIcon,
          title: this.$t('views.UserProfileView.ClaimedUrls'),
        },
      ];
    },
  },
  mounted() {
    this.reloadData();

    EventBus.$on('reloadData', () => {
      this.reloadData();
    });

    const interval = setInterval(() => this.reloadData(), 120 * 1000);
    this.$once('hook:beforeDestroy', () => clearInterval(interval));

    const { method, challenge, signature } = this.$route.query;
    if (!this.useSdkWallet && method && challenge && signature) {
      this.applyBackendChanges(method, { challenge, signature });
    }
  },
  methods: {
    openExplorer(address) {
      return this.explorerUrl + address;
    },
    resetEditedValues() {
      this.editMode = false;
      this.getProfile();
    },
    async applyBackendChanges(method, request) {
      const args = {
        sendProfileData: [request],
        deleteProfileImage: [this.account, request],
        setProfileImage: [this.account, request, false],
      }[method];
      if (!args) throw new Error(`Unknown method: ${method}`);
      await Backend[method](...args);
      this.resetEditedValues();
    },
    async backendAuth(method, challenge) {
      if (this.useSdkWallet) {
        const signature = await client.signMessage(challenge);
        this.applyBackendChanges(method, { challenge, signature });
      } else {
        const url = new URL(window.location);
        url.search = '';
        window.location = createDeepLinkUrl({
          type: 'sign-message',
          message: challenge,
          'x-success': `${url}?method=${method}&challenge=${challenge}&signature={signature}`,
        });
      }
    },
    async saveProfile() {
      const { challenge } = await Backend.sendProfileData({
        biography: this.profile.biography,
        author: this.account,
      });
      await this.backendAuth('sendProfileData', challenge);
    },
    async deleteAvatar() {
      const { challenge } = await Backend.deleteProfileImage(this.account);
      await this.backendAuth('deleteProfileImage', challenge);
    },
    async uploadAvatar(event) {
      const data = new FormData();
      data.append('name', 'image');
      data.append('image', event.target.files[0]);
      const { challenge } = await Backend.setProfileImage(this.account, data);
      await this.backendAuth('setProfileImage', challenge);
    },
    reloadData() {
      this.getProfile();
      Backend.getCacheUserStats(this.address).then((stats) => {
        this.userStats = stats;
      });
    },
    getProfile() {
      Backend.getCommentCountForAddress(this.address)
        .then((userComment) => {
          this.userCommentCount = userComment.count;
        })
        .catch(console.error);

      Backend.getProfile(this.address)
        .then((profile) => {
          if (!profile) {
            return;
          }
          this.profile = profile;
          this.profile.biography = this.profile.biography || '';
          this.$store.commit('setUserProfile', profile);
        })
        .catch(console.error);
    },
  },
};
</script>

<style lang="scss">
#file-input {
  display: none;
}

.profile__description {
  color: $tip_note_color;
  display: flex;
  flex-flow: wrap;
  margin: 0.5rem 1rem;

  textarea {
    background: no-repeat url('../assets/textEdit.svg') bottom 0.1rem right;
    border: 0;
    border-bottom: 1px solid $standard_font_color;
    color: $tip_note_color;
    display: flex;
    flex: 1 1 100%;
    resize: none;

    &::-webkit-scrollbar {
      display: none;
    }

    &:focus {
      background: no-repeat url('../assets/textEditFocus.svg') bottom 0.1rem right;
      border-bottom: 1px solid $secondary_color;
      outline: 0;
    }
  }

  .text-length {
    color: $light_font_color;
    flex-grow: 1;
    margin-top: 0.2rem;
    text-align: right;
  }
}

.profile__page {
  color: $tip_note_color;
  font-size: 0.75rem;

  .error {
    color: $red_color;
  }

  .edit__buttons {
    display: flex;
    flex-direction: column;
    position: absolute;
    right: 0.5rem;
    top: 0.75rem;

    .profile__button {
      margin-bottom: 0.6rem;
    }
  }

  .profile__button {
    border: 0;
    border-radius: 100%;
    cursor: pointer;
    flex-basis: 1.5rem;
    flex-grow: 0;
    height: 1.5rem;
    margin: 0;
    padding: 0;
    width: 1.5rem;

    img {
      width: 100%;
    }

    &:focus {
      outline: 0;
    }
  }

  .avatar__button {
    background: #2a9cffa8;
    bottom: 0;
    left: 0;
    position: absolute;

    .tip-user-text {
      display: none;
    }

    &:hover {
      background: #2a9cffcc;
    }
  }

  .delete_avatar__button {
    background: #ff49521e;
    display: none;
    position: absolute;
    right: -0.25rem;
    top: -0.25rem;

    .tip-user-text {
      display: none;
    }

    &:hover {
      background: #ff495242;
    }
  }

  & .profile__image:hover .delete_avatar__button {
    display: unset;
  }

  .edit__button,
  .cancel__button {
    background: #babac01e;

    &:hover {
      background: #babac042;
    }
  }

  .save__button {
    background: #67f7b8a8;

    &:hover {
      background: #67f7b8cc;
    }
  }

  .no-results {
    color: $standard_font_color;
    font-size: 0.75rem;
    text-align: center;
  }
}

.profile__header {
  align-items: flex-start;
  display: flex;
  padding: 1rem;
  position: relative;

  .joined {
    font-size: 0.7rem;
    margin-top: 0.5rem;

    time {
      color: $standard_font_color;
      margin-left: 0.2rem;
    }
  }
}

.tip_stats {
  display: grid;
  grid-template-columns: auto auto auto;
}

.tips_stats_block {
  background-color: $thumbnail_background_color;
  border-right: 1px solid $light_color;
  padding: 0.5rem 1rem;
}

.tips_stats_block:last-child {
  border: 0;
}

.stats {
  background-color: $thumbnail_background_color;
  border-top: 1px solid $light_color;
  display: flex;
  padding: 0.5rem 1rem;
  width: 100%;
}

.stat_block {
  margin-right: 2rem;

  img {
    height: 0.75rem;
  }
}

.stat-value {
  color: $tip_note_color;
  font-size: 0.75rem;
  vertical-align: middle;
}

.stat-title {
  color: $lighter_font_color;
  font-size: 0.7rem;
  vertical-align: middle;
}

.stat-value .currency-value {
  font-size: 80%;
}

.profile__section {
  background-color: $light_color;
  position: relative;
}

.profile__image {
  display: flex;
  margin-right: 0.5rem;
  position: relative;
  vertical-align: super;

  img.user-identicon,
  div.user-identicon svg {
    border-radius: 100%;
    height: 5.5rem;
    object-fit: cover;
    width: 5.5rem;
  }
}

.profile__info {
  display: flex;
  flex-direction: column;
  width: calc(100% - 8.5rem);

  .profile__username {
    color: $tip_note_color;
    display: block;
    font-size: 0.6rem;
    font-weight: 400;
    margin-bottom: 0;
    word-break: break-all;

    .chain {
      color: $standard_font_color;
      font-size: 0.9rem;
      font-weight: 500;
    }
  }
}

.profile__meta {
  background-color: $thumbnail_background_color;
  border-top-right-radius: 0.25rem;
  font-size: 0.6rem;
  margin: -0.5rem 0 -1rem 0;
  padding: 0;

  & > .row.mobile {
    display: none;
  }
}

@media only screen and (max-width: 768px) {
  .profile__page .profile__meta {
    border-top-right-radius: 0;
    margin-top: 0;
  }
}

@media screen and (max-width: 1024px) {
  .profile__page {
    .tips__container {
      padding: 0.15rem 0.5rem;
    }

    .edit__buttons {
      flex-direction: row;
      top: 0.4rem;
    }
  }

  .profile__header {
    padding: 1.4rem 1rem;
    white-space: nowrap;

    img.user-identicon,
    div.user-identicon svg {
      height: 5rem;
      object-fit: cover;
      width: 5rem;
    }
  }

  .profile__info {
    vertical-align: middle;
    width: calc(100% - 4.5rem);

    .profile__username {
      font-size: 0.55rem;
    }
  }

  .profile__page .avatar__button {
    bottom: -0.5rem;
  }

  .profile__page .delete_avatar__button {
    top: -0.5rem;
  }

  .profile__stats {
    display: grid;
    grid-template-columns: 8rem auto;
  }

  .tip_stats {
    grid-template-columns: auto;
    order: 2;
  }

  .tips_stats_block {
    border: 0;
    border-bottom: 1px solid $light_color;
  }

  .stats {
    border: 0;
    border-right: 1px solid $light_color;
    flex-direction: column;
    justify-content: space-around;
  }

  .stat_block {
    margin: 0;
  }
}
</style>
