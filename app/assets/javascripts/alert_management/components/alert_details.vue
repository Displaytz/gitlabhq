<script>
import * as Sentry from '@sentry/browser';
import {
  GlAlert,
  GlIcon,
  GlLoadingIcon,
  GlDropdown,
  GlDropdownItem,
  GlSprintf,
  GlTabs,
  GlTab,
  GlButton,
  GlTable,
} from '@gitlab/ui';
import createFlash from '~/flash';
import { s__ } from '~/locale';
import query from '../graphql/queries/details.query.graphql';
import { fetchPolicies } from '~/lib/graphql';
import TimeAgoTooltip from '~/vue_shared/components/time_ago_tooltip.vue';
import glFeatureFlagsMixin from '~/vue_shared/mixins/gl_feature_flags_mixin';
import { ALERTS_SEVERITY_LABELS } from '../constants';
import updateAlertStatus from '../graphql/mutations/update_alert_status.graphql';
import createIssueQuery from '../graphql/mutations/create_issue_from_alert.graphql';
import { visitUrl, joinPaths } from '~/lib/utils/url_utility';

export default {
  statuses: {
    TRIGGERED: s__('AlertManagement|Triggered'),
    ACKNOWLEDGED: s__('AlertManagement|Acknowledged'),
    RESOLVED: s__('AlertManagement|Resolved'),
  },
  i18n: {
    errorMsg: s__(
      'AlertManagement|There was an error displaying the alert. Please refresh the page to try again.',
    ),
    fullAlertDetailsTitle: s__('AlertManagement|Alert details'),
    overviewTitle: s__('AlertManagement|Overview'),
    reportedAt: s__('AlertManagement|Reported %{when}'),
    reportedAtWithTool: s__('AlertManagement|Reported %{when} by %{tool}'),
  },
  severityLabels: ALERTS_SEVERITY_LABELS,
  components: {
    GlAlert,
    GlIcon,
    GlLoadingIcon,
    GlSprintf,
    GlDropdown,
    GlDropdownItem,
    GlTab,
    GlTabs,
    GlButton,
    GlTable,
    TimeAgoTooltip,
  },
  mixins: [glFeatureFlagsMixin()],
  props: {
    alertId: {
      type: String,
      required: true,
    },
    projectPath: {
      type: String,
      required: true,
    },
    projectIssuesPath: {
      type: String,
      required: true,
    },
  },
  apollo: {
    alert: {
      fetchPolicy: fetchPolicies.CACHE_AND_NETWORK,
      query,
      variables() {
        return {
          fullPath: this.projectPath,
          alertId: this.alertId,
        };
      },
      update(data) {
        return data?.project?.alertManagementAlerts?.nodes?.[0] ?? null;
      },
      error(error) {
        this.errored = true;
        Sentry.captureException(error);
      },
    },
  },
  data() {
    return {
      alert: null,
      errored: false,
      isErrorDismissed: false,
      createIssueError: '',
      issueCreationInProgress: false,
    };
  },
  computed: {
    loading() {
      return this.$apollo.queries.alert.loading;
    },
    reportedAtMessage() {
      return this.alert?.monitoringTool
        ? this.$options.i18n.reportedAtWithTool
        : this.$options.i18n.reportedAt;
    },
    showErrorMsg() {
      return this.errored && !this.isErrorDismissed;
    },
  },
  methods: {
    dismissError() {
      this.isErrorDismissed = true;
    },
    updateAlertStatus(status) {
      this.$apollo
        .mutate({
          mutation: updateAlertStatus,
          variables: {
            iid: this.alertId,
            status: status.toUpperCase(),
            projectPath: this.projectPath,
          },
        })
        .catch(() => {
          createFlash(
            s__(
              'AlertManagement|There was an error while updating the status of the alert. Please try again.',
            ),
          );
        });
    },
    createIssue() {
      this.issueCreationInProgress = true;

      this.$apollo
        .mutate({
          mutation: createIssueQuery,
          variables: {
            iid: this.alert.iid,
            projectPath: this.projectPath,
          },
        })
        .then(({ data: { createAlertIssue: { errors, issue } } }) => {
          if (errors?.length) {
            [this.createIssueError] = errors;
            this.issueCreationInProgress = false;
          } else if (issue) {
            visitUrl(this.issuePath(issue.iid));
          }
        })
        .catch(error => {
          this.createIssueError = error;
          this.issueCreationInProgress = false;
        });
    },
    issuePath(issueId) {
      return joinPaths(this.projectIssuesPath, issueId);
    },
  },
};
</script>
<template>
  <div>
    <gl-alert v-if="showErrorMsg" variant="danger" @dismiss="dismissError">
      {{ $options.i18n.errorMsg }}
    </gl-alert>
    <gl-alert
      v-if="createIssueError"
      variant="danger"
      data-testid="issueCreationError"
      @dismiss="createIssueError = null"
    >
      {{ createIssueError }}
    </gl-alert>
    <div v-if="loading"><gl-loading-icon size="lg" class="gl-mt-5" /></div>
    <div v-if="alert" class="alert-management-details gl-relative">
      <div
        class="gl-display-flex gl-justify-content-space-between gl-align-items-baseline gl-px-1 py-3 py-md-4 gl-border-b-1 gl-border-b-gray-200 gl-border-b-solid flex-column flex-sm-row"
      >
        <div
          data-testid="alert-header"
          class="gl-display-flex gl-align-items-center gl-justify-content-center"
        >
          <div
            class="gl-display-inline-flex gl-align-items-center gl-justify-content-space-between"
          >
            <gl-icon
              class="gl-mr-3 align-middle"
              :size="12"
              :name="`severity-${alert.severity.toLowerCase()}`"
              :class="`icon-${alert.severity.toLowerCase()}`"
            />
            <strong>{{ $options.severityLabels[alert.severity] }}</strong>
          </div>
          <span class="mx-2">&bull;</span>
          <gl-sprintf :message="reportedAtMessage">
            <template #when>
              <time-ago-tooltip :time="alert.createdAt" class="gl-ml-3" />
            </template>
            <template #tool>{{ alert.monitoringTool }}</template>
          </gl-sprintf>
        </div>
        <div v-if="glFeatures.alertManagementCreateAlertIssue">
          <gl-button
            v-if="alert.issueIid"
            class="gl-mt-3 mt-sm-0 align-self-center align-self-sm-baseline alert-details-issue-button"
            data-testid="viewIssueBtn"
            :href="issuePath(alert.issueIid)"
            category="primary"
            variant="success"
          >
            {{ s__('AlertManagement|View issue') }}
          </gl-button>
          <gl-button
            v-else
            class="gl-mt-3 mt-sm-0 align-self-center align-self-sm-baseline alert-details-issue-button"
            data-testid="createIssueBtn"
            :loading="issueCreationInProgress"
            category="primary"
            variant="success"
            @click="createIssue()"
          >
            {{ s__('AlertManagement|Create issue') }}
          </gl-button>
        </div>
      </div>
      <div
        v-if="alert"
        class="gl-display-flex gl-justify-content-space-between gl-align-items-center"
      >
        <h2 data-testid="title">{{ alert.title }}</h2>
      </div>
      <gl-dropdown :text="$options.statuses[alert.status]" class="gl-absolute gl-right-0" right>
        <gl-dropdown-item
          v-for="(label, field) in $options.statuses"
          :key="field"
          data-testid="statusDropdownItem"
          class="gl-vertical-align-middle"
          @click="updateAlertStatus(label)"
        >
          <span class="d-flex">
            <gl-icon
              class="flex-shrink-0 append-right-4"
              :class="{ invisible: label.toUpperCase() !== alert.status }"
              name="mobile-issue-close"
            />
            {{ label }}
          </span>
        </gl-dropdown-item>
      </gl-dropdown>
      <gl-tabs v-if="alert" data-testid="alertDetailsTabs">
        <gl-tab data-testid="overviewTab" :title="$options.i18n.overviewTitle">
          <ul class="pl-4 mb-n1">
            <li v-if="alert.startedAt" class="my-2">
              <strong class="bold">{{ s__('AlertManagement|Start time') }}:</strong>
              <time-ago-tooltip data-testid="startTimeItem" :time="alert.startedAt" />
            </li>
            <li v-if="alert.eventCount" class="my-2">
              <strong class="bold">{{ s__('AlertManagement|Events') }}:</strong>
              <span data-testid="eventCount">{{ alert.eventCount }}</span>
            </li>
            <li v-if="alert.monitoringTool" class="my-2">
              <strong class="bold">{{ s__('AlertManagement|Tool') }}:</strong>
              <span data-testid="monitoringTool">{{ alert.monitoringTool }}</span>
            </li>
            <li v-if="alert.service" class="my-2">
              <strong class="bold">{{ s__('AlertManagement|Service') }}:</strong>
              <span data-testid="service">{{ alert.service }}</span>
            </li>
          </ul>
        </gl-tab>
        <gl-tab data-testid="fullDetailsTab" :title="$options.i18n.fullAlertDetailsTitle">
          <gl-table
            class="alert-management-details-table"
            :items="[{ key: 'Value', ...alert }]"
            :show-empty="true"
            :busy="loading"
            stacked
          >
            <template #empty>
              {{ s__('AlertManagement|No alert data to display.') }}
            </template>
            <template #table-busy>
              <gl-loading-icon size="lg" color="dark" class="mt-3" />
            </template>
          </gl-table>
        </gl-tab>
      </gl-tabs>
    </div>
  </div>
</template>
