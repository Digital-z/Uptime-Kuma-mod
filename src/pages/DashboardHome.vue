<template>
    <transition ref="tableContainer" name="slide-fade" appear>
        <div v-if="$route.name === 'DashboardHome'">
            <h1 class="mb-3">
                {{ $t("Quick Stats") }}
            </h1>

            <div class="shadow-box big-padding text-center mb-4">
                <div class="row">
                    <div class="col">
                        <h3>{{ $t("Up") }}</h3>
                        <!-- 将"正常"状态的数字始终显示为绿色 -->
                        <span
                            class="num"
                            :class="'text-success'"
                        >
                            {{ $root.stats.up }}
                        </span>
                    </div>
                    <div class="col">
                        <h3>{{ $t("Down") }}</h3>
                        <!-- 将"故障"状态的数字始终显示为红色 -->
                        <span
                            class="num"
                            :class="'text-danger'"
                        >
                            {{ $root.stats.down }}
                        </span>
                    </div>
                    <div class="col">
                        <h3>{{ $t("Maintenance") }}</h3>
                        <!-- 将"维护"状态的数字始终显示为蓝色 -->
                        <span
                            class="num"
                            :class="'text-primary'"
                        >
                            {{ $root.stats.maintenance }}
                        </span>
                    </div>
                    <div class="col">
                        <h3>{{ $t("Unknown") }}</h3>
                        <!-- 将"未知"状态的数字显示为深灰色 -->
                        <span class="num" style="color: #444444">{{ $root.stats.unknown }}</span>
                    </div>
                    <div class="col">
                        <h3>{{ $t("pauseDashboardHome") }}</h3>
                        <!-- 将"暂停"状态的数字显示为浅灰色 -->
                        <span class="num" style="color: #999999">{{ $root.stats.pause }}</span>
                    </div>
                </div>
            </div>

            <div class="shadow-box table-shadow-box" style="overflow-x: hidden">
                <table class="table table-borderless table-hover">
                    <thead>
                        <tr>
                            <th>{{ $t("Name") }}</th>
                            <th>{{ $t("Status") }}</th>
                            <th>{{ $t("DateTime") }}</th>
                            <th>{{ $t("Message") }}</th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr
                            v-for="(beat, index) in displayedRecords"
                            :key="index"
                            :class="{ 'shadow-box': $root.windowWidth <= 550 }"
                        >
                            <td class="name-column">
                                <router-link
                                    :to="`/dashboard/${beat.monitorID}`"
                                >{{ $root.monitorList[beat.monitorID]?.name }}</router-link>
                            </td>
                            <td><Status :status="beat.status" /></td>
                            <td :class="{ 'border-0': !beat.msg }">
                                <Datetime :value="beat.time" />
                            </td>
                            <td class="border-0">{{ beat.msg }}</td>
                        </tr>

                        <tr v-if="importantHeartBeatListLength === 0">
                            <td colspan="4">{{ $t("No important events") }}</td>
                        </tr>
                    </tbody>
                </table>

                <div class="d-flex justify-content-center kuma_pagination">
                    <pagination
                        v-model="page"
                        :records="importantHeartBeatListLength"
                        :per-page="perPage"
                        :options="paginationConfig"
                    />
                </div>
            </div>
        </div>
    </transition>
    <router-view ref="child" />
</template>

<script>
import Status from "../components/Status.vue";
import Datetime from "../components/Datetime.vue";
import Pagination from "v-pagination-3";

export default {
    components: {
        Datetime,
        Status,
        Pagination,
    },
    props: {
        calculatedHeight: {
            type: Number,
            default: 0,
        },
    },
    data() {
        return {
            page: 1,
            perPage: 25,
            initialPerPage: 25,
            paginationConfig: {
                hideCount: true,
                chunksNavigation: "scroll",
            },
            importantHeartBeatListLength: 0,
            displayedRecords: [],
        };
    },
    watch: {
        perPage() {
            this.$nextTick(() => {
                this.getImportantHeartbeatListPaged();
            });
        },

        page() {
            this.getImportantHeartbeatListPaged();
        },
    },

    mounted() {
        this.getImportantHeartbeatListLength();

        this.$root.emitter.on(
            "newImportantHeartbeat",
            this.onNewImportantHeartbeat
        );

        this.initialPerPage = this.perPage;

        window.addEventListener("resize", this.updatePerPage);
        this.updatePerPage();
    },

    beforeUnmount() {
        this.$root.emitter.off(
            "newImportantHeartbeat",
            this.onNewImportantHeartbeat
        );

        window.removeEventListener("resize", this.updatePerPage);
    },

    methods: {
        /**
         * Updates the displayed records when a new important heartbeat arrives.
         * @param {object} heartbeat - The heartbeat object received.
         * @returns {void}
         */
        onNewImportantHeartbeat(heartbeat) {
            if (this.page === 1) {
                this.displayedRecords.unshift(heartbeat);
                if (this.displayedRecords.length > this.perPage) {
                    this.displayedRecords.pop();
                }
                this.importantHeartBeatListLength += 1;
            }
        },

        /**
         * Retrieves the length of the important heartbeat list for all monitors.
         * @returns {void}
         */
        getImportantHeartbeatListLength() {
            this.$root
                .getSocket()
                .emit("monitorImportantHeartbeatListCount", null, (res) => {
                    if (res.ok) {
                        this.importantHeartBeatListLength = res.count;
                        this.getImportantHeartbeatListPaged();
                    }
                });
        },

        /**
         * Retrieves the important heartbeat list for the current page.
         * @returns {void}
         */
        getImportantHeartbeatListPaged() {
            const offset = (this.page - 1) * this.perPage;
            this.$root
                .getSocket()
                .emit(
                    "monitorImportantHeartbeatListPaged",
                    null,
                    offset,
                    this.perPage,
                    (res) => {
                        if (res.ok) {
                            this.displayedRecords = res.data;
                        }
                    }
                );
        },

        /**
         * Updates the number of items shown per page based on the available height.
         * @returns {void}
         */
        updatePerPage() {
            const tableContainer = this.$refs.tableContainer;
            const tableContainerHeight = tableContainer.offsetHeight;
            const availableHeight = window.innerHeight - tableContainerHeight;
            const additionalPerPage = Math.floor(availableHeight / 58);

            if (additionalPerPage > 0) {
                this.perPage = Math.max(
                    this.initialPerPage,
                    this.perPage + additionalPerPage
                );
            } else {
                this.perPage = this.initialPerPage;
            }
        },
    },
};
</script>

<style lang="scss" scoped>
@import "../assets/vars.scss";

/* 毛玻璃效果卡片 */
.shadow-box {
    background: $glass-bg-light !important; /* 使用半透明背景 */
    backdrop-filter: blur($glass-blur); /* 背景模糊效果 */
    -webkit-backdrop-filter: blur($glass-blur); /* Safari浏览器兼容 */
    border: 1px solid $glass-border-light !important; /* 使用半透明边框 */
    border-radius: 16px; /* 圆角边框 */
    box-shadow: $glass-shadow; /* 轻微阴影效果 */
}

/* 表格卡片 */
.table-shadow-box {
    padding: 10px;
    border-radius: 16px;
    overflow: hidden;
}

/* 数字展示样式 */
.num {
    font-size: 35px;
    font-weight: 900;
    display: block;
}

/* 大内边距样式 */
.big-padding {
    padding: 20px;
}

.dark {
    /* 暗色模式下的卡片样式 */
    .shadow-box {
        background: $glass-bg-dark !important;
        border: 1px solid $glass-border-dark !important;
    }

    /* 暗色模式下的表格 */
    table {
        color: $dark-font-color;
    }
}

.slide-fade-enter-active {
    transition: all 0.3s ease;
}

.slide-fade-leave-active {
    transition: all 0.3s cubic-bezier(1, 0.5, 0.8, 1);
}

.slide-fade-enter-from,
.slide-fade-leave-to {
    transform: translateY(10px);
    opacity: 0;
}

/* 适配移动端的样式调整 */
@media (max-width: 550px) {
    .table thead {
        display: none;
    }

    .table tbody {
        tr {
            display: flex;
            flex-direction: column;
            padding: 10px;
            margin-bottom: 10px;
            box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1),
                0 2px 4px -1px rgba(0, 0, 0, 0.06);

            td {
                a {
                    font-size: 16px;
                }
            }
        }
    }

    /* 名称列样式 */
    .name-column {
        border-bottom: 1px dashed rgba(0, 0, 0, 0.1) !important;
        font-weight: 500;

        .dark & {
            border-bottom: 1px dashed rgba(255, 255, 255, 0.1) !important;
        }
    }
}

/* 分页样式 */
.kuma_pagination {
    padding: 10px 0;
}
</style>
