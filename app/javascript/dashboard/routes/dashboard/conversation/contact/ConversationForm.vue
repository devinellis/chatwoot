<template>
  <form class="conversation--form" @submit.prevent="handleSubmit">
    <div v-if="showNoInboxAlert" class="callout warning">
      <p>
        {{ $t('NEW_CONVERSATION.NO_INBOX') }}
      </p>
    </div>
    <div v-else>
      <div class="row">
        <div class="columns">
          <label :class="{ error: $v.targetInbox.$error }">
            {{ $t('NEW_CONVERSATION.FORM.INBOX.LABEL') }}
            <select v-model="targetInbox">
              <option
                v-for="contactableInbox in inboxes"
                :key="contactableInbox.inbox.id"
                :value="contactableInbox"
              >
                {{ contactableInbox.inbox.name }}
              </option>
            </select>
            <span v-if="$v.targetInbox.$error" class="message">
              {{ $t('NEW_CONVERSATION.FORM.INBOX.ERROR') }}
            </span>
          </label>
        </div>
        <div class="columns">
          <label>
            {{ $t('NEW_CONVERSATION.FORM.TO.LABEL') }}
            <div class="contact-input">
              <thumbnail
                :src="contact.thumbnail"
                size="24px"
                :username="contact.name"
                :status="contact.availability_status"
              />
              <h4 class="text-block-title contact-name">
                {{ contact.name }}
              </h4>
            </div>
          </label>
        </div>
      </div>
      <div v-if="isAnEmailInbox" class="row">
        <div class="columns">
          <label :class="{ error: $v.subject.$error }">
            {{ $t('NEW_CONVERSATION.FORM.SUBJECT.LABEL') }}
            <input
              v-model="subject"
              type="text"
              :placeholder="$t('NEW_CONVERSATION.FORM.SUBJECT.PLACEHOLDER')"
              @input="$v.subject.$touch"
            />
            <span v-if="$v.subject.$error" class="message">
              {{ $t('NEW_CONVERSATION.FORM.SUBJECT.ERROR') }}
            </span>
          </label>
        </div>
      </div>
      <div class="row">
        <div class="columns">
          <div class="canned-response">
            <canned-response
              v-if="showCannedResponseMenu && hasSlashCommand"
              :search-key="cannedResponseSearchKey"
              @click="replaceTextWithCannedResponse"
            />
          </div>
          <div v-if="isAnEmailInbox || isAnWebWidgetInbox">
            <label>
              {{ $t('NEW_CONVERSATION.FORM.MESSAGE.LABEL') }}
              <reply-email-head
                v-if="isAnEmailInbox"
                :cc-emails.sync="ccEmails"
                :bcc-emails.sync="bccEmails"
              />
              <label class="editor-wrap">
                <woot-message-editor
                  v-model="message"
                  class="message-editor"
                  :class="{ editor_warning: $v.message.$error }"
                  :placeholder="$t('NEW_CONVERSATION.FORM.MESSAGE.PLACEHOLDER')"
                  @blur="$v.message.$touch"
                />
                <span v-if="$v.message.$error" class="editor-warning__message">
                  {{ $t('NEW_CONVERSATION.FORM.MESSAGE.ERROR') }}
                </span>
              </label>
            </label>
          </div>
          <label v-else :class="{ error: $v.message.$error }">
            {{ $t('NEW_CONVERSATION.FORM.MESSAGE.LABEL') }}
            <textarea
              v-model="message"
              class="message-input"
              type="text"
              :placeholder="$t('NEW_CONVERSATION.FORM.MESSAGE.PLACEHOLDER')"
              @input="$v.message.$touch"
            />
            <span v-if="$v.message.$error" class="message">
              {{ $t('NEW_CONVERSATION.FORM.MESSAGE.ERROR') }}
            </span>
          </label>
        </div>
      </div>
    </div>
    <div class="modal-footer">
      <button class="button clear" @click.prevent="onCancel">
        {{ $t('NEW_CONVERSATION.FORM.CANCEL') }}
      </button>
      <woot-button type="submit" :is-loading="conversationsUiFlags.isCreating">
        {{ $t('NEW_CONVERSATION.FORM.SUBMIT') }}
      </woot-button>
    </div>
  </form>
</template>

<script>
import { mapGetters } from 'vuex';
import Thumbnail from 'dashboard/components/widgets/Thumbnail';
import WootMessageEditor from 'dashboard/components/widgets/WootWriter/Editor';
import ReplyEmailHead from 'dashboard/components/widgets/conversation/ReplyEmailHead';
import CannedResponse from 'dashboard/components/widgets/conversation/CannedResponse.vue';

import alertMixin from 'shared/mixins/alertMixin';
import { INBOX_TYPES } from 'shared/mixins/inboxMixin';
import { ExceptionWithMessage } from 'shared/helpers/CustomErrors';
import { required, requiredIf } from 'vuelidate/lib/validators';

export default {
  components: {
    Thumbnail,
    WootMessageEditor,
    ReplyEmailHead,
    CannedResponse,
  },
  mixins: [alertMixin],
  props: {
    contact: {
      type: Object,
      default: () => ({}),
    },
    onSubmit: {
      type: Function,
      default: () => {},
    },
  },
  data() {
    return {
      name: '',
      subject: '',
      message: '',
      showCannedResponseMenu: false,
      cannedResponseSearchKey: '',
      selectedInbox: '',
      bccEmails: '',
      ccEmails: '',
    };
  },
  validations: {
    subject: {
      required: requiredIf('isAnEmailInbox'),
    },
    message: {
      required,
    },
    targetInbox: {
      required,
    },
  },
  computed: {
    ...mapGetters({
      uiFlags: 'contacts/getUIFlags',
      conversationsUiFlags: 'contactConversations/getUIFlags',
      currentUser: 'getCurrentUser',
    }),
    getNewConversation() {
      const payload = {
        inboxId: this.targetInbox.inbox.id,
        sourceId: this.targetInbox.source_id,
        contactId: this.contact.id,
        message: { content: this.message },
        mailSubject: this.subject,
        assigneeId: this.currentUser.id,
      };
      if (this.ccEmails) {
        payload.message.cc_emails = this.ccEmails;
      }

      if (this.bccEmails) {
        payload.message.bcc_emails = this.bccEmails;
      }
      return payload;
    },
    targetInbox: {
      get() {
        return this.selectedInbox || '';
      },
      set(value) {
        this.selectedInbox = value;
      },
    },
    showNoInboxAlert() {
      if (!this.contact.contactableInboxes) {
        return false;
      }
      return this.inboxes.length === 0 && !this.uiFlags.isFetchingInboxes;
    },
    inboxes() {
      return this.contact.contactableInboxes || [];
    },
    isAnEmailInbox() {
      return (
        this.selectedInbox &&
        this.selectedInbox.inbox.channel_type === INBOX_TYPES.EMAIL
      );
    },
    isAnWebWidgetInbox() {
      return (
        this.selectedInbox &&
        this.selectedInbox.inbox.channel_type === INBOX_TYPES.WEB
      );
    },
  },
  watch: {
    message(value) {
      this.hasSlashCommand = value[0] === '/';
      const hasNextWord = value.includes(' ');
      const isShortCodeActive = this.hasSlashCommand && !hasNextWord;
      if (isShortCodeActive) {
        this.cannedResponseSearchKey = value.substr(1, value.length);
        this.showCannedResponseMenu = true;
      } else {
        this.cannedResponseSearchKey = '';
        this.showCannedResponseMenu = false;
      }
    },
  },
  methods: {
    onCancel() {
      this.$emit('cancel');
    },
    onSuccess() {
      this.$emit('success');
    },
    async handleSubmit() {
      this.$v.$touch();
      if (this.$v.$invalid) {
        return;
      }
      try {
        const payload = this.getNewConversation;
        const data = await this.onSubmit(payload);
        const action = {
          type: 'link',
          to: `/app/accounts/${data.account_id}/conversations/${data.id}`,
          message: this.$t('NEW_CONVERSATION.FORM.GO_TO_CONVERSATION'),
        };
        this.onSuccess();
        this.showAlert(
          this.$t('NEW_CONVERSATION.FORM.SUCCESS_MESSAGE'),
          action
        );
      } catch (error) {
        if (error instanceof ExceptionWithMessage) {
          this.showAlert(error.data);
        } else {
          this.showAlert(this.$t('NEW_CONVERSATION.FORM.ERROR_MESSAGE'));
        }
      }
    },
    replaceTextWithCannedResponse(message) {
      setTimeout(() => {
        this.message = message;
      }, 50);
    },
  },
};
</script>

<style scoped lang="scss">
.conversation--form {
  padding: var(--space-normal) var(--space-large) var(--space-large);
}

.canned-response {
  position: relative;
  top: var(--space-medium);

  ::v-deep .mention--box {
    border-left: 1px solid var(--color-border);
    border-right: 1px solid var(--color-border);
  }
}

.input-group-label {
  font-size: var(--font-size-small);
}

.contact-input {
  display: flex;
  align-items: center;
  height: 3.9rem;
  background: var(--color-background-light);

  border: 1px solid var(--color-border);
  padding: var(--space-smaller) var(--space-small);
  border-radius: var(--border-radius-small);

  .contact-name {
    margin: 0;
    margin-left: var(--space-small);
  }
}

.message-input {
  min-height: 8rem;
}

.modal-footer {
  display: flex;
  justify-content: flex-end;
}
</style>
