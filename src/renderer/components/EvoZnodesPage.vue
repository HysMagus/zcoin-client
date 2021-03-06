<template>
  <div class="evo-znode-page znode-table-container">
    <div class="window-height znode-table-container">
      <input
        class="control"
        v-focus
        @input="onInput"
        placeholder="Filter by label, ip address or collateral address"
        v-model="filter"
        style="font-style:italic"
      />
      <input
        class="round"
        v-model="showAllZnodes"
        type="checkbox"
        name="showallznodes"
        :checked="false"
        @change="showAllZnodesChange(showAllZnodes)"
      />
      <label for="showallznodes">
        Show all Znodes.
      </label>
      <div style="text-align:right">
        <p>Node Count: {{ tableData.length }}</p>
      </div>
      <section v-scrollable class="znode-table-container animated-table">
        <vuetable
          ref="vuetable"
          :api-mode="false"
          :fields="tableFields"
          :data-manager="dataManager"
          pagination-path="pagination"
          @vuetable:loaded="onLoadingCompleted"
          :track-by="trackBy"
          :per-page="perPage"
          :sort-order="sortOrder"
          @vuetable:row-clicked="onExpandClicked"
          @vuetable:pagination-data="onPaginationData"
          detail-row-component="evo-znodes-details-page"
        >
          <div slot="label" slot-scope="props">
            {{ props.rowData.label }}
          </div>

          <div slot="collateralAddress" slot-scope="props">
            {{ props.rowData.collateralAddress }}
          </div>

          <div slot="nextPaymentBlock" slot-scope="props">
            {{ props.rowData.nextPaymentBlock }}
          </div>

          <div slot="statusdetails" slot-scope="props">
            <p>{{ props.rowData.status }}</p>
          </div>

          <div slot="status" slot-scope="props">
            <ZnodeStatusGreen v-show="props.rowData.status == 'ENABLED'" />
            <ZnodeStatusRed v-show="props.rowData.status != 'ENABLED'" />
          </div>

          <div slot="expand" slot-scope="props" :style="{ cursor: 'pointer' }">
            <ZnodeCollapseButton v-show="props.rowData.expand" />
            <ZnodeExpandButton v-show="!props.rowData.expand" />
          </div>
        </vuetable>

        <div class="znode-table-container">
          <animated-table-pagination
            ref="pagination"
            @vuetable-pagination:change-page="onChangePage"
          />
        </div>
      </section>
    </div>
  </div>
</template>

<script>
import { mapGetters } from "vuex";
import Vue from "vue";
import { Vuetable } from "vuetable-2";
import _ from "lodash";
import EvoZnodesDetailsPage from "./ZnodePage/EvoZnodesDetailsPage.vue";
import ZnodeStatusGreen from "@/components//Icons/ZnodeStatusGreen";
import ZnodeStatusRed from "@/components//Icons/ZnodeStatusRed";
import ZnodeCollapseButton from "@/components//Icons/ZnodeCollapseButton";
import ZnodeExpandButton from "@/components//Icons/ZnodeExpandButton";
import AnimatedTablePagination from "@/components/AnimatedTable/AnimatedTablePagination";
import { convertToSatoshi, convertToCoin } from "#/lib/convert";
Vue.component("evo-znodes-details-page", EvoZnodesDetailsPage);

const tableFields = [
  {
    name: "label",
    title: "Label",
    width: "22%",
  },
  {
    name: "collateralAddress",
    title: "Collateral Address",
    width: "37%",
  },
  {
    name: "nextPaymentBlock",
    title: "Next Payment (Block)",
    sortField: "nextPaymentBlock",
    width: "18%",
  },
  {
    name: "statusdetails",
    title: "Status",
    sortField: "statusdetails",
    width: "12%",
  },
  {
    name: "status",
    title: "",
    width: "5%",
  },
  {
    name: "expand",
    title: "",
    width: "3%",
  },
];

export default {
  name: "EvoZnodesPage",
  components: {
    Vuetable,
    AnimatedTablePagination,
    EvoZnodesDetailsPage,
    ZnodeStatusRed,
    ZnodeStatusGreen,
    ZnodeExpandButton,
    ZnodeCollapseButton,
  },
  props: {
    trackBy: {
      type: String,
      default: "proTxHash",
    },
    sortOrder: {
      type: Array,
      default: () => [
        {
          field: "statusdetails",
          direction: "asc",
        },
        {
          field: "nextPaymentBlock",
          direction: "desc",
        },
      ],
    },
    perPage: {
      type: Number,
      default: 8,
    },
    noDataMessage: {
      type: String,
      default: "No znodes",
    },
  },
  data() {
    return {
      tableFields,
      filter: "",
      showAllZnodes: false,
      tableData: [],
      oldFilter: "",
    };
  },
  mounted() {
    this.tableData = this.myZnodesTableData;
    this.$refs.vuetable.refresh();
  },

  computed: {
    ...mapGetters({
      masternodes: "Masternode/masternodes",
    }),

    filterString() {
      return this.filter;
    },

    shouldShowAllZnodes() {
      return this.showAllZnodes;
    },

    filteredZnodesTableData() {
      return [];
    },

    filteredMyZnodesTableData() {
      return [];
    },

    allZnodesTableData() {
      const tableData = [];
      for (const proTxHash of Object.keys(this.masternodes)) {
        let znodeObj = this.masternodes[proTxHash];
        let nextPay = znodeObj.state.nextPaymentHeight
          ? znodeObj.state.nextPaymentHeight
          : "SYNCING";
        if (znodeObj.state.status == "POSE_BANNED") {
          nextPay = "UNKNOWN";
        }
        tableData.push({
          proTxHash: proTxHash,
          label:
            znodeObj.label && znodeObj.label.length > 0
              ? znodeObj.label
              : "(unlabelled)",
          collateralAddress: znodeObj.collateralAddress,
          lastpaid: znodeObj.state.lastPaidHeight,
          nextPaymentBlock: nextPay,
          status: znodeObj.state.status ? znodeObj.state.status : "SYNCING",
          expand: false,
          service: znodeObj.state.service,
          ownerAddress: znodeObj.state.ownerAddress,
          PoSScore: znodeObj.state.PoSePenalty,
          registeredBlock: znodeObj.state.registeredHeight,
          paymentAddress: znodeObj.state.payoutAddress,
          isMine: znodeObj.wallet && znodeObj.wallet.hasMasternode
        });
      }
      return tableData;
    },

    myZnodesTableData() {
      const tableData = [];
      for (const znode of this.allZnodesTableData) {
        if (znode.isMine) {
          tableData.push(znode);
        }
      }
      return tableData;
    },

    filteredTableData() {
      return this.tableData;
    },
  },
  
  methods: {
    onCellClicked(data) {
      data.expand = !data.expand;
      this.$refs.vuetable.toggleDetailRow(data.proTxHash);
    },
    onExpandClicked(d) {
      this.onCellClicked(d.data);
    },

    async showAllZnodesChange(e) {
      this.oldFilter = "";
      this.applyFilter();
      this.$refs.vuetable.refresh();
    },

    onInput(event) {
      this.applyFilter();
      this.$refs.vuetable.refresh();
      this.oldFilter = this.filter;
    },

    applyFilter() {
      if (this.filter.length == 0) {
        if (this.showAllZnodes) {
          this.tableData = this.allZnodesTableData;
        } else {
          this.tableData = this.myZnodesTableData;
        }
        this.oldFilter = this.filter;
        return;
      }

      var tableRef = [];
      var tableOuts = [];
      if (this.showAllZnodes) {
        tableRef = this.allZnodesTableData;
      } else {
        tableRef = this.myZnodesTableData;
      }
      var newFilter = this.filter;
      if (this.oldFilter.length == 0 || !this.filter.includes(this.oldFilter)) {
        tableOuts = tableRef.filter(
          (e) =>
            (e.label && e.label.includes(newFilter)) ||
            e.service.includes(newFilter) ||
            e.collateralAddress.includes(newFilter)
        );
      } else {
        tableOuts = this.tableData;
        tableOuts = tableRef.filter(
          (e) =>
            (e.label && e.label.includes(newFilter)) ||
            e.service.includes(newFilter) ||
            e.collateralAddress.includes(newFilter)
        );
      }
      this.tableData = JSON.parse(JSON.stringify(tableOuts));
    },

    onPaginationData(paginationData) {
      this.$refs.pagination.setPaginationData(paginationData);
    },

    onChangePage(page) {
      this.rowTransition = "";
      this.$refs.vuetable.changePage(page);
      this.rowTransition = "fade";
    },

    collapseExpand() {
      let len = this.tableData.length;
      this.tableData.forEach((e) => {
        if (len <= 2) {
          if (!e.expand) {
            this.$refs.vuetable.toggleDetailRow(e.proTxHash);
            e.expand = true;
          }
        } else {
          if (e.expand) {
            this.$refs.vuetable.toggleDetailRow(e.proTxHash);
            e.expand = false;
          }
        }
      });
    },
    dataManager(sortOrder, pagination) {
      if (this.tableData.length < 1) {
        return {
          data: [],
        };
      }

      let local = this.tableData;
      const orderBy = sortOrder.length ? sortOrder : this.sortOrder;
      if (orderBy.length > 0) {
        local = _.orderBy(local, orderBy[0].sortField, orderBy[0].direction);
      }

      pagination = this.$refs.vuetable.makePagination(
        local.length,
        this.perPage
      );
      let from = pagination.from - 1;
      let to = from + this.perPage;
  
      return {
        pagination: pagination,
        data: _.slice(local, from, to),
      };
    },

    onLoadingCompleted() {
      this.collapseExpand();
    },
  },
};
</script>

<style lang="scss" scoped>
.control {
  display: flex;
  padding-right: 0;
  width: 32%;
}
.evo-znode-page {
  //display: grid;
  top: 2%;
  left: 10px;
  right: 9%;
  padding: 5%px;
  overflow-y: scroll;

  display: grid;
  box-sizing: border-box;

  .scrollable {
    position: relative;
    z-index: 20000;
    box-sizing: border-box;
    overflow: scroll;
    height: 1000vh;
  }
  & > div {
    height: 100%;
    //@include scrollbar-on-hover($color--comet-dark-mixed);
  }
}

.round {
  position: relative;
}

.round label {
  background-color: #fff;
  border: 1px solid #ccc;
  border-radius: 50%;
  cursor: pointer;
  height: 28px;
  left: 0;
  position: absolute;
  top: 0;
  width: 28px;
}

.round label:after {
  border: 2px solid #fff;
  border-top: none;
  border-right: none;
  content: "";
  height: 6px;
  left: 7px;
  opacity: 0;
  position: absolute;
  top: 8px;
  transform: rotate(-45deg);
  width: 12px;
}

.round input[type="checkbox"] {
  visibility: hidden;
}

.round input[type="checkbox"]:checked + label {
  background-color: #66bb6a;
  border-color: #66bb6a;
}

.round input[type="checkbox"]:checked + label:after {
  opacity: 1;
}

.znode-table-container {
  position: relative;
}
</style>
