<template>
  <v-dialog v-model="showCreate" persistent max-width="800px">
    <template v-slot:activator="{ on }">
      <v-btn icon v-on="on"><v-icon>add</v-icon></v-btn>
    </template>
    <v-card>
      <v-card-title>
        <span class="headline">Create Search Filter</span>
      </v-card-title>
      <v-stepper v-model="step">
        <v-stepper-header>
          <v-stepper-step :complete="step > 1" step="1" editable>
            Filter
          </v-stepper-step>
          <v-divider></v-divider>

          <v-stepper-step :complete="step > 2" step="2" editable>
            Preview
          </v-stepper-step>
          <v-divider></v-divider>

          <v-stepper-step step="3" editable>
            Save
          </v-stepper-step>
        </v-stepper-header>

        <v-stepper-items>
          <v-stepper-content step="1">
            <v-card>
              <v-card-text>
                <v-tabs color="primary" right>
                  <v-tab>Basic</v-tab>
                  <v-tab>Advanced</v-tab>
                  <v-tab-item>
                    <v-list dense>
                      <v-list-item>
                        <v-list-item-content>
                          <tag-filter-combobox v-model="filters.tags" label="Tags" />
                        </v-list-item-content>
                      </v-list-item>
                      <v-list-item>
                        <v-list-item-content>
                          <incident-type-combobox v-model="filters.incident_type" />
                        </v-list-item-content>
                      </v-list-item>
                      <v-list-item>
                        <v-list-item-content>
                          <incident-priority-combobox v-model="filters.incident_priority" />
                        </v-list-item-content>
                      </v-list-item>
                      <v-list-item>
                        <v-list-item-content>
                          <incident-status-multi-select v-model="filters.status" />
                        </v-list-item-content>
                      </v-list-item>
                    </v-list>
                  </v-tab-item>
                  <v-tab-item>
                    <div style="height: 400px">
                      <advanced-editor v-model="expression"></advanced-editor>
                    </div>
                  </v-tab-item>
                </v-tabs>
              </v-card-text>
              <v-card-actions>
                <v-btn color="info" @click="step = 2">
                  Continue
                </v-btn>

                <v-btn @click="closeCreateDialog()" text>
                  Cancel
                </v-btn>
              </v-card-actions>
            </v-card>
          </v-stepper-content>

          <v-stepper-content step="2">
            <v-card>
              <v-card-text>
                Examples matching your filter:
                <v-data-table
                  hide-default-footer
                  :headers="previewFields"
                  :items="previewRows.items"
                  :loading="previewRowsLoading"
                >
                  <template v-slot:item.incident_priority.name="{ item }">
                    <incident-priority :priority="item.incident_priority.name" />
                  </template>
                  <template v-slot:item.status="{ item }">
                    <incident-status :status="item.status" :id="item.id" />
                  </template>
                </v-data-table>
              </v-card-text>
              <v-card-actions>
                <v-btn color="info" @click="step = 3" :loading="loading">
                  Continue
                </v-btn>
                <v-btn @click="closeCreateDialog()" text>
                  Cancel
                </v-btn>
              </v-card-actions>
            </v-card>
          </v-stepper-content>
          <v-stepper-content step="3">
            <ValidationObserver disabled v-slot="{ invalid, validated }">
              <v-card>
                <v-card-text>
                  Provide a name and description for your filter.
                  <ValidationProvider name="Name" rules="required" immediate>
                    <v-text-field
                      v-model="name"
                      label="Name"
                      hint="A name for your saved search."
                      slot-scope="{ errors, valid }"
                      :error-messages="errors"
                      :success="valid"
                      clearable
                      required
                    />
                  </ValidationProvider>
                  <ValidationProvider name="Description" rules="required" immediate>
                    <v-textarea
                      v-model="description"
                      label="Description"
                      hint="A short description."
                      slot-scope="{ errors, valid }"
                      :error-messages="errors"
                      :success="valid"
                      clearable
                      auto-grow
                      required
                    />
                  </ValidationProvider>
                </v-card-text>
                <v-card-actions>
                  <v-spacer />
                  <v-btn @click="closeCreateDialog()" text>
                    Cancel
                  </v-btn>
                  <v-btn
                    color="info"
                    @click="save('incident')"
                    :loading="loading"
                    :disabled="invalid || !validated"
                  >
                    Save
                  </v-btn>
                </v-card-actions>
              </v-card>
            </ValidationObserver>
          </v-stepper-content>
        </v-stepper-items>
      </v-stepper>
    </v-card>
  </v-dialog>
</template>

<script>
import { mapFields } from "vuex-map-fields"
import { ValidationObserver, ValidationProvider, extend } from "vee-validate"
import { required } from "vee-validate/dist/rules"

import { mapActions } from "vuex"

import IncidentApi from "@/incident/api"
import IncidentStatusMultiSelect from "@/incident/IncidentStatusMultiSelect.vue"
import TagFilterCombobox from "@/tag/TagFilterCombobox.vue"
import IncidentTypeCombobox from "@/incident_type/IncidentTypeCombobox.vue"
import IncidentPriorityCombobox from "@/incident_priority/IncidentPriorityCombobox.vue"
import IncidentStatus from "@/incident/IncidentStatus.vue"
import IncidentPriority from "@/incident/IncidentPriority.vue"
import AdvancedEditor from "@/search/AdvancedEditor.vue"
import SearchUtils from "@/search/utils"

extend("required", {
  ...required,
  message: "This field is required"
})

export default {
  name: "SearchFilterCreateDialog",
  data() {
    return {
      previewFields: [
        { text: "Name", value: "name", sortable: false },
        { text: "Title", value: "title", sortable: false },
        { text: "Status", value: "status", sortable: false },
        { text: "Incident Type", value: "incident_type.name", sortable: false },
        { text: "Incident Priority", value: "incident_priority.name", sortable: false }
      ],
      step: 1,
      previewRows: {
        items: [],
        total: null
      },
      previewRowsLoading: false,
      filters: {
        incident_type: null,
        incident_priority: null,
        status: null,
        tag: null
      }
    }
  },
  components: {
    ValidationObserver,
    ValidationProvider,
    TagFilterCombobox,
    IncidentTypeCombobox,
    IncidentPriorityCombobox,
    IncidentStatusMultiSelect,
    IncidentStatus,
    IncidentPriority,
    AdvancedEditor
  },
  computed: {
    ...mapFields("search", [
      "selected",
      "selected.description",
      "selected.expression",
      "selected.name",
      "loading",
      "dialogs.showCreate"
    ])
  },

  methods: {
    ...mapActions("search", ["closeCreateDialog", "save"]),

    saveFilter() {
      // reset local data
      Object.assign(this.$data, this.$options.data.apply(this))
      this.save("incident")
      this.$emit("input", this.selected)
    },

    getPreviewData() {
      let params = { filter: JSON.stringify(this.expression) }
      this.previewRowsLoading = "error"
      return IncidentApi.getAll(params).then(response => {
        this.previewRows = response.data
        this.previewRowsLoading = false
      })
    }
  },
  mounted() {
    this.type = "incident"
    this.getPreviewData()
    this.$watch(
      vm => [
        vm.filters.incident_type,
        vm.filters.incident_priority,
        vm.filters.status,
        vm.filters.tag
      ],
      () => {
        this.expression = SearchUtils.createFilterExpression(this.filters)
        this.getPreviewData()
      }
    )
  }
}
</script>
