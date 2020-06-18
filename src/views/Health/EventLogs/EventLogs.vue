<template>
  <b-container fluid="xl">
    <page-title />
    <b-row class="mb-3">
      <b-col md="6" lg="7" xl="5" offset-md="6" offset-lg="5" offset-xl="7">
        <table-date-filter @change="onChangeDateTimeFilter" />
      </b-col>
    </b-row>
    <b-row>
      <b-col class="text-right">
        <table-filter :filters="tableFilters" @filterChange="onFilterChange" />
      </b-col>
    </b-row>
    <b-row>
      <b-col>
        <table-toolbar
          ref="toolbar"
          :selected-items-count="selectedRows.length"
          :actions="batchActions"
          @clearSelected="clearSelectedRows($refs.table)"
          @batchAction="onBatchAction"
        >
          <template v-slot:export>
            <table-toolbar-export
              :data="batchExportData"
              :file-name="$t('appPageTitle.eventLogs')"
            />
          </template>
        </table-toolbar>
        <b-table
          id="table-event-logs"
          ref="table"
          selectable
          no-select-on-click
          sort-icon-left
          no-sort-reset
          sort-desc
          show-empty
          sort-by="date"
          :fields="fields"
          :items="filteredLogs"
          :sort-compare="onSortCompare"
          :empty-text="$t('pageEventLogs.table.emptyMessage')"
          :per-page="perPage"
          :current-page="currentPage"
          @row-selected="onRowSelected($event, filteredLogs.length)"
        >
          <!-- Checkbox column -->
          <template v-slot:head(checkbox)>
            <b-form-checkbox
              v-model="tableHeaderCheckboxModel"
              :indeterminate="tableHeaderCheckboxIndeterminate"
              @change="onChangeHeaderCheckbox($refs.table)"
            />
          </template>
          <template v-slot:cell(checkbox)="row">
            <b-form-checkbox
              v-model="row.rowSelected"
              @change="toggleSelectRow($refs.table, row.index)"
            />
          </template>

          <!-- Severity column -->
          <template v-slot:cell(severity)="{ value }">
            <status-icon :status="getStatus(value)" />
            {{ value }}
          </template>

          <!-- Date column -->
          <template v-slot:cell(date)="{ value }">
            {{ value | formatDate }} {{ value | formatTime }}
          </template>

          <!-- Actions column -->
          <template v-slot:cell(actions)="{ item }">
            <table-row-action
              v-for="(action, index) in item.actions"
              :key="index"
              :value="action.value"
              :title="action.title"
              :row-data="item"
              :export-name="item.id"
              @click:tableAction="onTableRowAction($event, item)"
            >
              <template v-slot:icon>
                <icon-export v-if="action.value === 'export'" />
                <icon-trashcan v-if="action.value === 'delete'" />
              </template>
            </table-row-action>
          </template>
        </b-table>
      </b-col>
    </b-row>

    <!-- Table pagination -->
    <b-row>
      <b-col class="d-md-flex justify-content-between">
        <b-form-group
          class="table-pagination-select"
          :label="$t('global.table.itemsPerPage')"
          label-for="pagination-items-per-page"
        >
          <b-form-select
            id="pagination-items-per-page"
            v-model="perPage"
            :options="itemsPerPageOptions"
          />
        </b-form-group>
        <b-pagination
          v-model="currentPage"
          first-number
          last-number
          :per-page="perPage"
          :total-rows="getTotalRowCount(filteredLogs.length)"
          aria-controls="table-event-logs"
        />
      </b-col>
    </b-row>
  </b-container>
</template>

<script>
import IconTrashcan from '@carbon/icons-vue/es/trash-can/20';
import IconExport from '@carbon/icons-vue/es/export/20';
import { omit } from 'lodash';

import PageTitle from '@/components/Global/PageTitle';
import StatusIcon from '@/components/Global/StatusIcon';
import TableDateFilter from '@/components/Global/TableDateFilter';
import TableFilter from '@/components/Global/TableFilter';
import TableRowAction from '@/components/Global/TableRowAction';
import TableToolbar from '@/components/Global/TableToolbar';
import TableToolbarExport from '@/components/Global/TableToolbarExport';

import LoadingBarMixin from '@/components/Mixins/LoadingBarMixin';
import TableFilterMixin from '@/components/Mixins/TableFilterMixin';
import BVPaginationMixin from '@/components/Mixins/BVPaginationMixin';
import BVTableSelectableMixin from '@/components/Mixins/BVTableSelectableMixin';
import BVToastMixin from '@/components/Mixins/BVToastMixin';

const SEVERITY = ['OK', 'Warning', 'Critical'];

export default {
  components: {
    IconExport,
    IconTrashcan,
    PageTitle,
    StatusIcon,
    TableFilter,
    TableRowAction,
    TableToolbar,
    TableToolbarExport,
    TableDateFilter
  },
  mixins: [
    BVPaginationMixin,
    BVTableSelectableMixin,
    BVToastMixin,
    LoadingBarMixin,
    TableFilterMixin
  ],
  data() {
    return {
      fields: [
        {
          key: 'checkbox',
          sortable: false
        },
        {
          key: 'id',
          label: this.$t('pageEventLogs.table.id'),
          sortable: true
        },
        {
          key: 'severity',
          label: this.$t('pageEventLogs.table.severity'),
          sortable: true
        },
        {
          key: 'type',
          label: this.$t('pageEventLogs.table.type'),
          sortable: true
        },
        {
          key: 'date',
          label: this.$t('pageEventLogs.table.date'),
          sortable: true
        },
        {
          key: 'description',
          label: this.$t('pageEventLogs.table.description')
        },
        {
          key: 'actions',
          sortable: false,
          label: '',
          tdClass: 'text-right'
        }
      ],
      tableFilters: [
        {
          label: this.$t('pageEventLogs.table.severity'),
          values: SEVERITY
        }
      ],
      activeFilters: [],
      batchActions: [
        {
          value: 'delete',
          label: this.$t('global.action.delete')
        }
      ],
      filterStartDate: null,
      filterEndDate: null
    };
  },
  computed: {
    allLogs() {
      return this.$store.getters['eventLog/allEvents'].map(event => {
        return {
          ...event,
          actions: [
            {
              value: 'export',
              title: this.$t('global.action.export')
            },
            {
              value: 'delete',
              title: this.$t('global.action.delete')
            }
          ]
        };
      });
    },
    batchExportData() {
      return this.selectedRows.map(row => omit(row, 'actions'));
    },
    filteredLogsByDate() {
      return this.getFilteredTableDataByDate(
        this.allLogs,
        this.filterStartDate,
        this.filterEndDate
      );
    },
    filteredLogs() {
      return this.getFilteredTableData(
        this.filteredLogsByDate,
        this.activeFilters
      );
    }
  },
  created() {
    this.startLoader();
    this.$store
      .dispatch('eventLog/getEventLogData')
      .finally(() => this.endLoader());
  },
  beforeRouteLeave(to, from, next) {
    // Hide loader if the user navigates to another page
    // before request is fulfilled.
    this.hideLoader();
    next();
  },
  methods: {
    getStatus(serverity) {
      switch (serverity) {
        case SEVERITY[2]:
          return 'danger';
        case SEVERITY[1]:
          return 'warning';
        case SEVERITY[0]:
          return 'success';
        default:
          return '';
      }
    },
    deleteLogs(uris) {
      this.$store.dispatch('eventLog/deleteEventLogs', uris).then(messages => {
        messages.forEach(({ type, message }) => {
          if (type === 'success') {
            this.successToast(message);
          } else if (type === 'error') {
            this.errorToast(message);
          }
        });
      });
    },
    onFilterChange({ activeFilters }) {
      this.activeFilters = activeFilters;
    },
    onSortCompare(a, b, key) {
      if (key === 'severity') {
        return SEVERITY.indexOf(a.status) - SEVERITY.indexOf(b.status);
      }
    },
    onTableRowAction(action, { uri }) {
      if (action === 'delete') {
        this.$bvModal
          .msgBoxConfirm(this.$tc('pageEventLogs.modal.deleteMessage'), {
            title: this.$tc('pageEventLogs.modal.deleteTitle'),
            okTitle: this.$t('global.action.delete')
          })
          .then(deleteConfirmed => {
            if (deleteConfirmed) this.deleteLogs([uri]);
          });
      }
    },
    onBatchAction(action) {
      if (action === 'delete') {
        const uris = this.selectedRows.map(row => row.uri);
        this.$bvModal
          .msgBoxConfirm(
            this.$tc(
              'pageEventLogs.modal.deleteMessage',
              this.selectedRows.length
            ),
            {
              title: this.$tc(
                'pageEventLogs.modal.deleteTitle',
                this.selectedRows.length
              ),
              okTitle: this.$t('global.action.delete')
            }
          )
          .then(deleteConfirmed => {
            if (deleteConfirmed) this.deleteLogs(uris);
          });
      }
    },
    onChangeDateTimeFilter({ fromDate, toDate }) {
      this.filterStartDate = fromDate;
      this.filterEndDate = toDate;
    }
  }
};
</script>