<template>
    <!-- ============ Body content start ============= -->
    <div class="main-content">
        <div v-if="loading" class="loading_page spinner spinner-primary mr-3"></div>
        <div>
            <!-- warehouse -->
            <b-row>
                <!-- Stock Alert -->
                <div class="col-md-12">
                    <div class="card mb-30">
                        <div class="card-body p-2">
                            <h5 class="card-title border-bottom p-3 mb-2">{{ $t('ProductExpiryReport') }}</h5>

                            <vue-good-table
                                :columns="columns_stock"
                                styleClass="order-table vgt-table mb-3"
                                row-style-class="text-left"
                                :rows="sortedStockAlerts"
                                :search-options="{
                                    enabled: true,
                                    placeholder: 'Search for something...'
                                }"
                                :pagination-options="{
                                    enabled: true,
                                    perPage: 10
                                }"
                            >
                                <template slot="table-row" slot-scope="props">
                                    <span v-if="props.column.field == 'stock_alert'" class="badge badge-outline-danger">
                                        {{ props.row.stock_alert }}
                                    </span>
                                    <span v-else-if="props.column.field == 'exp_date'">
                                        <span :class="getExpiryClass(props.row.exp_date)">
                                            {{ props.row.exp_date ? props.row.exp_date : 'N/A' }}
                                        </span>
                                    </span>
                                    <span v-else-if="props.column.field == 'pr_no'">
                                        <span v-if="props.row.pr_no">
                                            <router-link :to="'/app/purchases/detail/' + props.row.purchase_id">
                                                {{ props.row.pr_no }}
                                            </router-link>
                                        </span>
                                        <span v-else>
                                            N/A
                                        </span>
                                    </span>
                                </template>
                            </vue-good-table>
                        </div>
                    </div>
                </div>
            </b-row>
        </div>
    </div>
    <!-- ============ Body content End ============= -->
</template>

<script>
import { mapGetters } from "vuex";
import ECharts from "vue-echarts/components/ECharts.vue";

// import ECharts modules manually to reduce bundle size
import "echarts/lib/chart/pie";
import "echarts/lib/chart/bar";
import "echarts/lib/chart/line";
import "echarts/lib/component/tooltip";
import "echarts/lib/component/legend";

export default {
    components: {
        "v-chart": ECharts,
    },
    metaInfo: {
        title: "Dashboard",
    },
    data() {
        return {
            sales: [],
            warehouses: [],
            warehouse_id: "",
            stock_alerts: [],
            report_today: {
                revenue: 0,
                today_purchases: 0,
                today_sales: 0,
                return_sales: 0,
                return_purchases: 0,
            },
            products: [],
            CurrentMonth: "",
            loading: true,
            echartSales: {},
            echartProduct: {},
            echartCustomer: {},
            echartPayment: {},
        };
    },
    computed: {
        ...mapGetters(["currentUserPermissions", "currentUser"]),
        columns_stock() {
            return [
                {
                    label: this.$t("ProductCode"),
                    field: "code",
                    tdClass: "text-left",
                    thClass: "text-left",
                    sortable: false,
                },
                {
                    label: this.$t("ProductName"),
                    field: "name",
                    tdClass: "text-left",
                    thClass: "text-left",
                    sortable: false,
                },
                {
                    label: this.$t("warehouse"),
                    field: "warehouse",
                    tdClass: "text-left",
                    thClass: "text-left",
                    sortable: false,
                },
                {
                    label: this.$t("Quantity"),
                    field: "quantity",
                    tdClass: "text-left",
                    thClass: "text-left",
                    sortable: false,
                },
                {
                    label: this.$t("PrNo"),
                    field: "pr_no",
                    tdClass: "text-left",
                    thClass: "text-left",
                    sortable: false,
                },
                {
                    label: this.$t("ExpiryDate"),
                    field: "exp_date",
                    tdClass: "text-left",
                    thClass: "text-left",
                    sortable: false,
                },
                {
                    label: this.$t("AlertQuantity"),
                    field: "stock_alert",
                    tdClass: "text-left",
                    thClass: "text-left",
                    sortable: false,
                },
            ];
        },
        sortedStockAlerts() {
            return this.stock_alerts.slice().sort((a, b) => {
                const currentDate = new Date();
                const expiryDateA = a.exp_date ? new Date(a.exp_date) : null;
                const expiryDateB = b.exp_date ? new Date(b.exp_date) : null;

                // If both are null, they are equal
                if (expiryDateA === null && expiryDateB === null) return 0;

                // If only one is null, it should come last
                if (expiryDateA === null) return 1;
                if (expiryDateB === null) return -1;

                const dayDiffA = (expiryDateA - currentDate) / (1000 * 3600 * 24);
                const dayDiffB = (expiryDateB - currentDate) / (1000 * 3600 * 24);

                if (dayDiffA <= 0 && dayDiffB > 0) return -1; // a is expired, b is not
                if (dayDiffA > 0 && dayDiffB <= 0) return 1; // b is expired, a is not
                if (dayDiffA <= 60 && dayDiffB > 60) return -1; // a expires within 2 months, b does not
                if (dayDiffA > 60 && dayDiffB <= 60) return 1; // b expires within 2 months, a does not

                return dayDiffA - dayDiffB; // sort by expiry date if both are in the future
            });
        },
    },
    methods: {
        Selected_Warehouse(value) {
            if (value === null) {
                this.warehouse_id = "";
            }
            this.all_dashboard_data();
        },

        all_dashboard_data() {
            axios
                .get("/dashboard_data?warehouse_id=" + this.warehouse_id)
                .then((response) => {
                    const responseData = response.data;

                    this.stock_alerts = responseData.report_dashboard.original.stock_alert_1;
                    this.products = responseData.report_dashboard.original.products;
                    this.sales = responseData.report_dashboard.original.last_sales;
                    this.loading = false;
                })
                .catch((response) => {});
        },

        getExpiryClass(expDate) {
            const currentDate = new Date();
            const expiryDate = new Date(expDate);
            const timeDiff = expiryDate - currentDate;
            const dayDiff = Math.ceil(timeDiff / (1000 * 3600 * 24));

            if (dayDiff <= 0) {
                return "text-danger"; // red for expired
            } else if (dayDiff <= 60) {
                return "text-warning"; // yellow for less than or equal to 2 months
            } else {
                return ""; // no color for more than 2 months
            }
        },
    },
    async mounted() {
        await this.all_dashboard_data();
        this.GetMonth();
    },
};
</script>
