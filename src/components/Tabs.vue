<template>
<div>
  <el-tabs v-model="selectedTabName" type="card" closable @tab-remove="removeTab">
    <el-tab-pane
      v-for="(item) in tabs"
      :key="item.name"
      :name="item.name">
      <span slot="label" :title="item.title">
        <i :class="iconNameByComponent(item.component)"></i>
        <span>{{ item.label }}</span>
      </span>
      <Status :client='item.client' v-if="item.component === 'status'"></Status>
      <CliTab :client='item.client' v-else-if="item.component === 'cli'"></CliTab>
      <KeyDetail :client='item.client' v-else :redisKey="item.redisKey" :keyType="item.keyType"></KeyDetail>
    </el-tab-pane>
  </el-tabs>
</div>
</template>

<script>
import Status from '@/components/Status';
import CliTab from '@/components/CliTab';
import KeyDetail from '@/components/KeyDetail';

export default {
  data() {
    return {
      selectedTabName: '',
      tabs: [],
    };
  },
  components: { Status, KeyDetail, CliTab },
  created() {
    // key clicked
    this.$bus.$on('clickedKey', (client, key, newTab = false) => {
      this.addKeyTab(client, key, newTab);
    });

    // open status tab
    this.$bus.$on('openStatus', (client, tabName) => {
      this.addStatusTab(client, tabName);
    });

    // open cli tab
    this.$bus.$on('openCli', (client, tabName) => {
      this.addCliTab(client, tabName);
    });

    // remove pre tab
    this.$bus.$on('removePreTab', () => {
      this.removeTab(this.selectedTabName);
    });

    // remove all tab
    this.$bus.$on('removeAllTab', () => {
      this.tabs = [];
    });

    // add new key tab
    this.$bus.$on('addNewKey', (client, type) => {
      this.addTab(this.initKeyTabItem(client, '', type), true);
    });
  },
  methods: {
    removeTab(removeName) {
      const { tabs } = this;
      let nextSelectTab;

      if (this.selectedTabName == removeName) {
        tabs.forEach((tab, index) => {
          if (tab.name == removeName) {
            nextSelectTab = tabs[index + 1] || tabs[index - 1];
          }
        });
      }

      nextSelectTab && (this.selectedTabName = nextSelectTab.name);
      this.tabs = this.tabs.filter(tab => tab.name !== removeName);
    },
    addStatusTab(client, tabName, newTab = false) {
      const newTabItem = {
        name: `status_${tabName}`,
        label: this.$util.cutString(tabName),
        title: tabName,
        client: client,
        component: 'status',
      }

      this.addTab(newTabItem, newTab);
    },
    addCliTab(client, tabName, newTab = true) {
      const newTabItem = {
        name: `cli_${tabName}`,
        label: this.$util.cutString(tabName),
        title: tabName,
        client: client,
        component: 'cli',
      }

      this.addTab(newTabItem, newTab);
    },
    addKeyTab(client, key, newTab = false) {
      client.typeAsync(key).then((type) => {
        // key not exists
        if (type === 'none') {
          this.$message.error({
            message: `${key} ${this.$t('message.key_not_exists')}`,
            duration: 1000,
          });

          return;
        }

        this.addTab(this.initKeyTabItem(client, key, type), newTab);
      });
    },
    initKeyTabItem(client, key, type) {
      const cutString = this.$util.cutString;
      const dbIndex = client.selected_db ? client.selected_db : 0;
      const connectionName = client.options.connection_name;

      const label = `${cutString(key)} | ${cutString(connectionName)} | DB${dbIndex}`;
      const name  = `${key} | ${connectionName} | DB${dbIndex}`;

      return {
        name: name, label: label, title: name, client: client, component: 'key',
        redisKey: key, keyType: type,
      };
    },
    addTab(newTabItem, newTab = false) {
      let exists = false;

      this.tabs.map((item) => {
        (item.name === newTabItem.name) && (exists = true);
      });

      // if exists, select directly
      if (exists) {
        this.selectedTabName = newTabItem.name;
        return;
      }

      // new tab append to tail
      if (newTab) {
        this.tabs.push(newTabItem);
      }

      // open tab on previous selected key tab
      // or append to tail if previous tab is cli\status
      else {
        let replaced = false;

        this.tabs = this.tabs.map((item) => {
          // replace the selected tab with new tab item
          if (item.name === this.selectedTabName && item.component === 'key') {
            replaced = true;
            return newTabItem;
          }

          return item;
        });

        // pre tab is preserve tab, append to tail
        !replaced && (this.tabs.push(newTabItem));
      }

      this.selectedTabName = newTabItem.name;
    },
    iconNameByComponent(component) {
      const map = {
        cli: 'fa fa-terminal',
        status: 'el-icon-info',
      };

      const icon = map[component];

      return icon ? icon : 'fa fa-key';
    },
  },
};
</script>
