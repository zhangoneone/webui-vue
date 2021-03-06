<template>
  <b-container fluid="xl">
    <page-title />
    <b-row>
      <b-col md="8">
        <page-section
          :section-title="$t('pageSecuritySettings.networkServices')"
        >
          <b-row class="setting-section">
            <b-col class="d-flex align-items-center justify-content-between">
              <dl class="mr-3 w-75">
                <dt>{{ $t('pageSecuritySettings.ssh') }}</dt>
                <dd>
                  {{ $t('pageSecuritySettings.sshDescription') }}
                </dd>
              </dl>
              <b-form-checkbox
                id="sshSwitch"
                v-model="sshProtocolState"
                data-test-id="securitySettings-checkbox-switchSshProtocol"
                switch
                @change="changeSshProtocolState"
              >
                <span class="sr-only">
                  {{ $t('pageSecuritySettings.ssh') }}
                </span>
                <span v-if="sshProtocolState">
                  {{ $t('global.status.enabled') }}
                </span>
                <span v-else>{{ $t('global.status.disabled') }}</span>
              </b-form-checkbox>
            </b-col>
          </b-row>
          <b-row class="setting-section">
            <b-col class="d-flex align-items-center justify-content-between">
              <dl class="mt-3 mr-3 w-75">
                <dt>{{ $t('pageSecuritySettings.ipmi') }}</dt>
                <dd>
                  {{ $t('pageSecuritySettings.ipmiDescription') }}
                </dd>
              </dl>
              <b-form-checkbox
                id="ipmiSwitch"
                v-model="ipmiProtocolState"
                data-test-id="securitySettings-checkbox-switchIpmiProtocol"
                switch
                @change="changeIpmiProtocolState"
              >
                <span class="sr-only">
                  {{ $t('pageSecuritySettings.ipmi') }}
                </span>
                <span v-if="ipmiProtocolState">
                  {{ $t('global.status.enabled') }}
                </span>
                <span v-else>{{ $t('global.status.disabled') }}</span>
              </b-form-checkbox>
            </b-col>
          </b-row>
        </page-section>
      </b-col>
    </b-row>
  </b-container>
</template>

<script>
import PageSection from '@/components/Global/PageSection';
import PageTitle from '@/components/Global/PageTitle';

import LoadingBarMixin from '@/components/Mixins/LoadingBarMixin';
import BVToastMixin from '@/components/Mixins/BVToastMixin';

export default {
  name: 'SecuritySettings',
  components: { PageTitle, PageSection },
  mixins: [LoadingBarMixin, BVToastMixin],
  beforeRouteLeave(to, from, next) {
    this.hideLoader();
    next();
  },
  computed: {
    sshProtocolState: {
      get() {
        return this.$store.getters['securitySettings/sshProtocolEnabled'];
      },
      set(newValue) {
        return newValue;
      },
    },
    ipmiProtocolState: {
      get() {
        return this.$store.getters['securitySettings/ipmiProtocolEnabled'];
      },
      set(newValue) {
        return newValue;
      },
    },
  },
  created() {
    this.startLoader();
    this.$store
      .dispatch('securitySettings/getNetworkProtocolStatus')
      .finally(() => this.endLoader());
  },
  methods: {
    changeIpmiProtocolState(state) {
      this.$store
        .dispatch('securitySettings/saveIpmiProtocolState', state)
        .then((message) => this.successToast(message))
        .catch(({ message }) => this.errorToast(message));
    },
    changeSshProtocolState(state) {
      this.$store
        .dispatch('securitySettings/saveSshProtocolState', state)
        .then((message) => this.successToast(message))
        .catch(({ message }) => this.errorToast(message));
    },
  },
};
</script>

<style lang="scss" scoped>
.setting-section {
  border-bottom: 1px solid gray('300');
}
</style>
