<template>
  <v-dialog :value="show" lazy persistent max-width="400" class="p-share-upload-dialog" @keydown.esc="cancel">
    <v-card raised elevation="24">
      <v-card-title primary-title class="pb-0">
        <v-layout row wrap>
          <v-flex xs8>
            <h3 class="headline mb-0">
              <translate>WebDAV Upload</translate>
            </h3>
          </v-flex>
          <v-flex xs4 text-xs-right>
            <v-btn icon flat dark color="secondary-dark" class="ma-0" @click.stop="setup">
              <v-icon>cloud</v-icon>
            </v-btn>
          </v-flex>
        </v-layout>
      </v-card-title>
      <v-card-text class="pt-0">
        <v-layout row wrap>
          <v-flex xs12 text-xs-left class="pt-2">
            <v-select
                v-model="account"
                color="secondary-dark" hide-details hide-no-data
                flat
                :label="$gettext('Account')"
                item-text="AccName"
                item-value="ID"
                return-object
                :disabled="loading || noAccounts"
                :items="accounts"
                @change="onChange">
            </v-select>
          </v-flex>
          <v-flex xs12 text-xs-left class="pt-2">
            <v-autocomplete
                v-model="path"
                color="secondary-dark" hide-details hide-no-data
                flat
                browser-autocomplete="off"
                hint="Folder"
                :search-input.sync="search"
                :items="pathItems"
                :loading="loading"
                :disabled="loading || noAccounts"
                item-text="abs"
                item-value="abs"
                :label="$gettext('Folder')"
            >
            </v-autocomplete>
          </v-flex>
          <v-flex xs12 text-xs-right class="pt-4">
            <v-btn depressed color="secondary-light" class="action-cancel ml-0 mt-0 mb-0 mr-2" @click.stop="cancel">
              <translate>Cancel</translate>
            </v-btn>
            <v-btn v-if="noAccounts" :disabled="isPublic && !isDemo" color="primary-button" depressed dark
                   class="action-setup ma-0" @click.stop="setup">
              <translate>Setup</translate>
            </v-btn>
            <v-btn v-else :disabled="noAccounts" color="primary-button" depressed dark
                   class="action-upload ma-0" @click.stop="confirm">
              <translate>Upload</translate>
            </v-btn>
          </v-flex>
        </v-layout>
      </v-card-text>
    </v-card>
  </v-dialog>
</template>
<script>
import Account from "model/account";
import Selection from "common/selection";

export default {
  name: 'PShareUploadDialog',
  props: {
    show: Boolean,
    items: {
      type: Object,
      default: null,
    },
    model: {
      type: Object,
      default: null,
    }
  },
  data() {
    return {
      isDemo: this.$config.get("demo"),
      isPublic: this.$config.get("public"),
      noAccounts: false,
      loading: true,
      search: null,
      account: {},
      accounts: [],
      selection: new Selection({}),
      path: "/",
      paths: [
        {"abs": "/"}
      ],
      pathItems: [],
      newPath: "",
    };
  },
  watch: {
    search(q) {
      if (this.loading) return;

      const exists = this.paths.findIndex((p) => p.value === q);

      if (exists !== -1 || !q) {
        this.pathItems = this.paths;
        this.newPath = "";
      } else {
        this.newPath = q;
        this.pathItems = this.paths.concat([{"abs": q}]);
      }
    },
    show: function (show) {
      if (show) {
        this.load();
      } else if (this.selection) {
        this.selection.clear();
      }
    }
  },
  methods: {
    cancel() {
      this.$emit('cancel');
    },
    setup() {
      this.$router.push({name: "settings_sync"});
    },
    confirm() {
      if (this.noAccounts) {
        this.$notify.warn(this.$gettext('No servers configured.'));
        return;
      } else if (this.loading) {
        this.$notify.wait();
        return;
      }

      this.loading = true;
      this.account.Share(this.selection, this.path).then(
        (files) => {
          this.loading = false;

          if (files.length === 1) {
            this.$notify.success(this.$gettext("One file uploaded"));
          } else {
            this.$notify.success(this.$gettextInterpolate(this.$gettext("%{n} files uploaded"), {n: files.length}));
          }

          this.$emit('confirm', this.account);
        }
      ).catch(() => this.loading = false);
    },
    onChange() {
      this.paths = [{"abs": "/"}];

      this.loading = true;
      this.account.Folders().then(p => {
        for (let i = 0; i < p.length; i++) {
          this.paths.push(p[i]);
        }

        this.pathItems = [...this.paths];
        this.path = this.account.SharePath;
      }).finally(() => this.loading = false);
    },
    load() {
      this.loading = true;

      this.selection.clear().addItems(this.items);

      if (this.selection.isEmpty()) {
        this.selection.addModel(this.model);
      }

      if (this.selection.isEmpty()) {
        this.loading = false;
        this.$emit('cancel');
        return;
      }

      const params = {
        share: true,
        count: 1000,
        offset: 0,
      };

      Account.search(params).then(response => {
        if (!response.models.length) {
          this.noAccounts = true;
          this.loading = false;
        } else {
          this.account = response.models[0];
          this.accounts = response.models;
          this.onChange();
        }
      }).catch(() => this.loading = false);
    },
  },
};
</script>
