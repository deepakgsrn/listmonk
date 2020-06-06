<template>
  <section class="campaign">
    <header class="columns">
      <div class="column is-8">
        <p class="tags">
          <b-tag v-if="isEditing" :class="data.status">{{ data.status }}</b-tag>
          <b-tag v-if="data.type === 'optin'" :class="data.type">{{ data.type }}</b-tag>
          <span v-if="isEditing" class="has-text-grey-light is-size-7">
            ID: {{ data.id }} / UUID: {{ data.uuid }}
          </span>
        </p>
        <h4 v-if="isEditing" class="title is-4">{{ data.name }}</h4>
        <h4 v-else class="title is-4">New campaign</h4>
      </div>

      <div class="column">
        <div class="buttons" v-if="isEditing && canEdit">
          <b-button @click="onSubmit" :loading="loading.campaigns"
            type="is-primary" icon-left="content-save-outline">Save changes</b-button>

          <b-button v-if="canStart" @click="startCampaign" :loading="loading.campaigns"
            type="is-primary" icon-left="rocket-launch-outline">
              Start campaign
          </b-button>
          <b-button v-if="canSchedule" @click="startCampaign" :loading="loading.campaigns"
            type="is-primary" icon-left="clock-start">
              Schedule campaign
          </b-button>
        </div>
      </div>
    </header>

    <b-loading :active="loading.campaigns"></b-loading>

    <b-tabs type="is-boxed" :animated="false" v-model="activeTab" @change="onTabChange">
      <b-tab-item label="Campaign" icon="rocket-launch-outline">
        <section class="wrap">
          <form @submit.prevent="onSubmit">
            <b-field label="Name">
              <b-input :maxlength="200" :ref="'focus'" v-model="form.name" :disabled="!canEdit"
                placeholder="Name" required></b-input>
            </b-field>

            <b-field label="Subject">
              <b-input :maxlength="200" v-model="form.subject" :disabled="!canEdit"
                placeholder="Subject" required></b-input>
            </b-field>

            <b-field label="From address">
              <b-input :maxlength="200" v-model="form.fromEmail" :disabled="!canEdit"
                placeholder="Your Name <noreply@yoursite.com>" required></b-input>
            </b-field>

            <list-selector
              v-model="form.lists"
              :selected="form.lists"
              :all="lists.results"
              :disabled="!canEdit"
              label="Lists"
              placeholder="Lists to send to"
            ></list-selector>

            <b-field label="Template">
              <b-select placeholder="Template" v-model="form.templateId" :disabled="!canEdit"
                required>
                <option v-for="t in templates" :value="t.id" :key="t.id">{{ t.name }}</option>
              </b-select>
            </b-field>

            <b-field label="Tags">
              <b-taginput v-model="form.tags" :disabled="!canEdit"
                ellipsis icon="tag" placeholder="Tags"></b-taginput>
            </b-field>
            <hr />

            <b-field label="Send later?">
                <b-switch v-model="form.sendLater" :disabled="!canEdit"></b-switch>
            </b-field>

            <b-field v-if="form.sendLater" label="Send at">
              <b-datetimepicker
                v-model="form.sendAtDate"
                :disabled="!canEdit"
                placeholder="Date and time"
                icon="calendar-today"
                :timepicker="{ hourFormat: '24' }"
                :datetime-formatter="formatDateTime"
                horizontal-time-picker>
              </b-datetimepicker>
            </b-field>
            <hr />

            <b-field v-if="isNew">
              <b-button native-type="submit" type="is-primary"
                :loading="loading.campaigns">Continue</b-button>
            </b-field>
          </form>
        </section>
      </b-tab-item><!-- campaign -->

      <b-tab-item label="Content" icon="text" :disabled="isNew">
        <section class="wrap">
          <editor
            v-model="form.content"
            :contentType="data.contentType"
            :body="data.body"
            :disabled="!canEdit"
          />
        </section>
      </b-tab-item><!-- content -->
    </b-tabs>
  </section>
</template>

<script>
import Vue from 'vue';
import { mapState } from 'vuex';
import dayjs from 'dayjs';
import ListSelector from '../components/ListSelector.vue';
import Editor from '../components/Editor.vue';

Vue.component('list-selector', ListSelector);
Vue.component('editor', Editor);

export default Vue.extend({
  name: 'Campaign',

  data() {
    return {
      isNew: false,
      isEditing: false,
      activeTab: 0,

      data: {},

      // Binds form input values.
      form: {
        name: '',
        subject: '',
        fromEmail: '',
        templateId: 0,
        lists: [],
        tags: [],
        sendAt: null,
        content: { contentType: 'richtext', body: '' },

        // Parsed Date() version of send_at from the API.
        sendAtDate: null,
        sendLater: false,
      },
    };
  },

  methods: {
    formatDateTime(s) {
      return dayjs(s).format('YYYY-MM-DD HH:mm');
    },

    onTabChange() {
      // document.location.href = `#${index}`;
    },

    getCampaign(id) {
      this.$api.getCampaign(id).then((r) => {
        this.data = r.data;
        this.form = { ...this.form, ...r.data };

        if (r.data.sendAt !== null) {
          this.form.sendLater = true;
          this.form.sendAtDate = dayjs(r.data.sendAt).toDate();
        }
      });
    },

    onSubmit() {
      if (this.isNew) {
        this.createCampaign();
      } else {
        this.updateCampaign();
      }
    },

    createCampaign() {
      console.log(this.form);
      const data = {
        name: this.form.name,
        subject: this.form.subject,
        lists: this.form.lists.map((l) => l.id),
        from_email: this.form.fromEmail,
        content_type: 'richtext',
        messenger: 'email',
        type: 'regular',
        tags: this.form.tags,
        template_id: this.form.templateId,
        // body: this.form.body,
      };

      this.$api.createCampaign(data).then((r) => {
        this.$router.push({ name: 'campaign', params: { id: r.data.id } });

        this.data = r.data;
        this.isEditing = true;
        this.isNew = false;
        this.activeTab = 1;
      });
      return false;
    },

    async updateCampaign() {
      const data = {
        name: this.form.name,
        subject: this.form.subject,
        lists: this.form.lists.map((l) => l.id),
        from_email: this.form.fromEmail,
        messenger: 'email',
        type: 'regular',
        tags: this.form.tags,
        send_later: this.form.sendLater,
        send_at: this.form.sendLater ? this.form.sendAtDate : null,
        template_id: this.form.templateId,
        content_type: this.form.content.contentType,
        body: this.form.content.body,
      };

      // This promise is used by startCampaign to first save before starting.
      return new Promise((resolve) => {
        this.$api.updateCampaign(this.data.id, data).then((resp) => {
          this.data = resp.data;
          this.$buefy.toast.open({
            message: `'${resp.data.name}' updated`,
            type: 'is-success',
            queue: false,
          });
          resolve();
        });
      });
    },

    // Starts or schedule a campaign.
    startCampaign() {
      let fn;
      let msg = '';
      if (this.canStart) {
        fn = this.$api.startCampaign;
        msg = 'Are you sure? Campaign properties cannot be changed once it starts.';
      } else if (this.canSchedule) {
        fn = this.$api.scheduleCampaign;
        msg = 'Are you sure?';
      } else {
        return;
      }

      this.$buefy.dialog.confirm({
        scroll: 'keep',
        message: msg,
        onConfirm: () => {
          // First save the campaign.
          this.updateCampaign().then(() => {
            // Then start/schedule it.
            fn(this.data.id).then(() => {
              this.$router.push({
                name: 'campaigns',
                query: { id: this.data.id },
                hash: `#${this.data.id}`,
              });
            });
          });
        },
      });
    },
  },

  computed: {
    ...mapState(['lists', 'templates', 'loading']),

    canEdit() {
      return this.isNew
        || this.data.status === 'draft' || this.data.status === 'scheduled';
    },

    canSchedule() {
      return this.data.status === 'draft' && this.data.sendAt;
    },

    canStart() {
      return this.data.status === 'draft' && !this.data.sendAt;
    },
  },

  mounted() {
    const { id } = this.$route.params;


    // New campaign.
    if (id === 'new') {
      this.isNew = true;
    } else {
      const intID = parseInt(id, 10);
      if (intID <= 0 || Number.isNaN(intID)) {
        this.$buefy.toast.open({
          message: 'Invalid campaign',
          type: 'is-danger',
          queue: false,
        });
        return;
      }

      this.isEditing = true;
    }

    // Get templates list.
    this.$api.getTemplates().then((r) => {
      if (r.data.length > 0) {
        if (!this.form.templateId) {
          this.form.templateId = r.data[0].id;
        }
      }
    });

    // Fetch campaign.
    if (this.isEditing) {
      this.getCampaign(id);
    }

    this.$nextTick(() => {
      this.$refs.focus.focus();
    });
  },
});
</script>
