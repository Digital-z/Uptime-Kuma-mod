<template>
    <div class="shadow-box mb-3" :style="boxStyle">
        <div class="list-header">
            <div class="header-top">
                <button
                    class="btn btn-outline-normal ms-2"
                    :class="{ active: selectMode }"
                    type="button"
                    @click="selectMode = !selectMode"
                >
                    {{ $t("Select") }}
                </button>

                <div class="search-wrapper">
                    <a v-if="searchText == ''" class="search-icon">
                        <font-awesome-icon icon="search" />
                    </a>
                    <a
                        v-if="searchText != ''"
                        class="search-icon"
                        @click="clearSearchText"
                    >
                        <font-awesome-icon icon="times" />
                    </a>
                    <form>
                        <input
                            v-model="searchText"
                            class="form-control search-input"
                            :placeholder="$t('Search...')"
                            :aria-label="$t('Search monitored sites')"
                            autocomplete="off"
                        />
                    </form>
                </div>
            </div>
            <div class="header-filter">
                <MonitorListFilter
                    :filterState="filterState"
                    @update-filter="updateFilter"
                />
            </div>

            <!-- Selection Controls -->
            <div v-if="selectMode" class="selection-controls px-2 pt-2">
                <input
                    v-model="selectAll"
                    class="form-check-input select-input"
                    type="checkbox"
                    :aria-label="$t('Select/Deselect All')"
                />

                <button class="btn-outline-normal" @click="pauseDialog">
                    <font-awesome-icon icon="pause" size="sm" />
                    {{ $t("Pause") }}
                </button>
                <button class="btn-outline-normal" @click="resumeSelected">
                    <font-awesome-icon icon="play" size="sm" />
                    {{ $t("Resume") }}
                </button>

                <span v-if="selectedMonitorCount > 0">
                    {{ $t("selectedMonitorCount", [selectedMonitorCount]) }}
                </span>
            </div>
        </div>
        <div
            ref="monitorList"
            class="monitor-list"
            :class="{ scrollbar: scrollbar }"
            :style="monitorListStyle"
            data-testid="monitor-list"
        >
            <div
                v-if="Object.keys($root.monitorList).length === 0"
                class="text-center mt-3"
            >
                {{ $t("No Monitors, please") }}
                <router-link to="/add">{{ $t("add one") }}</router-link>
            </div>

            <MonitorListItem
                v-for="(item, index) in sortedMonitorList"
                :key="index"
                :monitor="item"
                :isSelectMode="selectMode"
                :isSelected="isSelected"
                :select="select"
                :deselect="deselect"
                :filter-func="filterFunc"
                :sort-func="sortFunc"
            />
        </div>
    </div>

    <Confirm
        ref="confirmPause"
        :yes-text="$t('Yes')"
        :no-text="$t('No')"
        @yes="pauseSelected"
    >
        {{ $t("pauseMonitorMsg") }}
    </Confirm>
</template>

<script>
import Confirm from "../components/Confirm.vue";
import MonitorListItem from "../components/MonitorListItem.vue";
import MonitorListFilter from "./MonitorListFilter.vue";
import { getMonitorRelativeURL } from "../util.ts";

export default {
    components: {
        Confirm,
        MonitorListItem,
        MonitorListFilter,
    },
    props: {
        /** Should the scrollbar be shown */
        scrollbar: {
            type: Boolean,
        },
    },
    data() {
        return {
            searchText: "",
            selectMode: false,
            selectAll: false,
            disableSelectAllWatcher: false,
            selectedMonitors: {},
            windowTop: 0,
            filterState: {
                status: null,
                active: null,
                tags: null,
            },
        };
    },
    computed: {
        /**
         * Improve the sticky appearance of the list by increasing its
         * height as user scrolls down.
         * Not used on mobile.
         * @returns {object} Style for monitor list
         */
        boxStyle() {
            if (window.innerWidth > 550) {
                return {
                    height: `calc(100vh - 160px + ${this.windowTop}px)`,
                };
            } else {
                return {
                    height: "calc(100vh - 160px)",
                };
            }
        },

        /**
         * Returns a sorted list of monitors based on the applied filters and search text.
         * @returns {Array} The sorted list of monitors.
         */
        sortedMonitorList() {
            let result = Object.values(this.$root.monitorList);

            result = result.filter((monitor) => {
                // The root list does not show children
                if (monitor.parent !== null) {
                    return false;
                }
                return true;
            });

            result = result.filter(this.filterFunc);

            result.sort(this.sortFunc);

            return result;
        },

        isDarkTheme() {
            return document.body.classList.contains("dark");
        },

        monitorListStyle() {
            let listHeaderHeight = 107;

            if (this.selectMode) {
                listHeaderHeight += 42;
            }

            return {
                height: `calc(100% - ${listHeaderHeight}px)`,
            };
        },

        selectedMonitorCount() {
            return Object.keys(this.selectedMonitors).length;
        },

        /**
         * Determines if any filters are active.
         * @returns {boolean} True if any filter is active, false otherwise.
         */
        filtersActive() {
            return (
                this.filterState.status != null ||
                this.filterState.active != null ||
                this.filterState.tags != null ||
                this.searchText !== ""
            );
        },
    },
    watch: {
        searchText() {
            for (let monitor of this.sortedMonitorList) {
                if (!this.selectedMonitors[monitor.id]) {
                    if (this.selectAll) {
                        this.disableSelectAllWatcher = true;
                        this.selectAll = false;
                    }
                    break;
                }
            }
        },
        selectAll() {
            if (!this.disableSelectAllWatcher) {
                this.selectedMonitors = {};

                if (this.selectAll) {
                    // 选择所有可见的监控项（包括子项）
                    const selectAllVisibleMonitors = (monitorList) => {
                        monitorList.forEach((item) => {
                            // 选择当前项
                            this.selectedMonitors[item.id] = true;

                            // 如果有子项且未折叠，递归选择子项
                            const children = Object.values(
                                this.$root.monitorList
                            ).filter((m) => m.parent === item.id);
                            if (children.length > 0) {
                                selectAllVisibleMonitors(children);
                            }
                        });
                    };

                    // 从根监控项开始选择
                    selectAllVisibleMonitors(this.sortedMonitorList);
                }
            } else {
                this.disableSelectAllWatcher = false;
            }
        },
        selectMode() {
            if (!this.selectMode) {
                this.selectAll = false;
                this.selectedMonitors = {};
            }
        },
    },
    mounted() {
        window.addEventListener("scroll", this.onScroll);
    },
    beforeUnmount() {
        window.removeEventListener("scroll", this.onScroll);
    },
    methods: {
        /**
         * Handle user scroll
         * @returns {void}
         */
        onScroll() {
            if (window.top.scrollY <= 133) {
                this.windowTop = window.top.scrollY;
            } else {
                this.windowTop = 133;
            }
        },
        /**
         * Get URL of monitor
         * @param {number} id ID of monitor
         * @returns {string} Relative URL of monitor
         */
        monitorURL(id) {
            return getMonitorRelativeURL(id);
        },
        /**
         * Clear the search bar
         * @returns {void}
         */
        clearSearchText() {
            this.searchText = "";
        },
        /**
         * Update the MonitorList Filter
         * @param {object} newFilter Object with new filter
         * @returns {void}
         */
        updateFilter(newFilter) {
            this.filterState = newFilter;
        },
        /**
         * Deselect a monitor
         * @param {number} id ID of monitor
         * @returns {void}
         */
        deselect(id) {
            delete this.selectedMonitors[id];
        },
        /**
         * Select a monitor
         * @param {number} id ID of monitor
         * @returns {void}
         */
        select(id) {
            this.selectedMonitors[id] = true;
        },
        /**
         * Determine if monitor is selected
         * @param {number} id ID of monitor
         * @returns {bool} Is the monitor selected?
         */
        isSelected(id) {
            return id in this.selectedMonitors;
        },
        /**
         * Disable select mode and reset selection
         * @returns {void}
         */
        cancelSelectMode() {
            this.selectMode = false;
            this.selectedMonitors = {};
        },
        /**
         * Show dialog to confirm pause
         * @returns {void}
         */
        pauseDialog() {
            this.$refs.confirmPause.show();
        },
        /**
         * Pause each selected monitor
         * @returns {void}
         */
        pauseSelected() {
            Object.keys(this.selectedMonitors)
                .filter((id) => this.$root.monitorList[id].active)
                .forEach((id) =>
                    this.$root.getSocket().emit("pauseMonitor", id, () => {})
                );

            this.cancelSelectMode();
        },
        /**
         * Resume each selected monitor
         * @returns {void}
         */
        resumeSelected() {
            Object.keys(this.selectedMonitors)
                .filter((id) => !this.$root.monitorList[id].active)
                .forEach((id) =>
                    this.$root.getSocket().emit("resumeMonitor", id, () => {})
                );

            this.cancelSelectMode();
        },
        /**
         * Whether a monitor should be displayed based on the filters
         * @param {object} monitor Monitor to check
         * @returns {boolean} Should the monitor be displayed
         */
        filterFunc(monitor) {
            // Group monitors bypass filter if at least 1 of children matched
            if (monitor.type === "group") {
                const children = Object.values(this.$root.monitorList).filter(
                    (m) => m.parent === monitor.id
                );
                if (
                    children.some((child, index, children) =>
                        this.filterFunc(child)
                    )
                ) {
                    return true;
                }
            }

            // filter by search text
            // finds monitor name, tag name or tag value
            let searchTextMatch = true;
            if (this.searchText !== "") {
                const loweredSearchText = this.searchText.toLowerCase();
                searchTextMatch =
                    monitor.name.toLowerCase().includes(loweredSearchText) ||
                    monitor.tags.find(
                        (tag) =>
                            tag.name
                                .toLowerCase()
                                .includes(loweredSearchText) ||
                            tag.value?.toLowerCase().includes(loweredSearchText)
                    );
            }

            // filter by status
            let statusMatch = true;
            if (
                this.filterState.status != null &&
                this.filterState.status.length > 0
            ) {
                if (
                    monitor.id in this.$root.lastHeartbeatList &&
                    this.$root.lastHeartbeatList[monitor.id]
                ) {
                    monitor.status =
                        this.$root.lastHeartbeatList[monitor.id].status;
                }
                statusMatch = this.filterState.status.includes(monitor.status);
            }

            // filter by active
            let activeMatch = true;
            if (
                this.filterState.active != null &&
                this.filterState.active.length > 0
            ) {
                activeMatch = this.filterState.active.includes(monitor.active);
            }

            // filter by tags
            let tagsMatch = true;
            if (
                this.filterState.tags != null &&
                this.filterState.tags.length > 0
            ) {
                tagsMatch =
                    monitor.tags
                        .map((tag) => tag.tag_id) // convert to array of tag IDs
                        .filter((monitorTagId) =>
                            this.filterState.tags.includes(monitorTagId)
                        ).length > 0; // perform Array Intersaction between filter and monitor's tags
            }

            return searchTextMatch && statusMatch && activeMatch && tagsMatch;
        },
        /**
         * Function used in Array.sort to order monitors in a list.
         * @param {*} m1 monitor 1
         * @param {*} m2 monitor 2
         * @returns {number} -1, 0 or 1
         */
        sortFunc(m1, m2) {
            if (m1.active !== m2.active) {
                if (m1.active === false) {
                    return 1;
                }

                if (m2.active === false) {
                    return -1;
                }
            }

            if (m1.weight !== m2.weight) {
                if (m1.weight > m2.weight) {
                    return -1;
                }

                if (m1.weight < m2.weight) {
                    return 1;
                }
            }

            return m1.name.localeCompare(m2.name);
        },
    },
};
</script>

<style lang="scss" scoped>
@import "../assets/vars.scss";

/* 列表头部样式 */
.list-header {
    .dark & {
        color: $dark-font-color;
    }

    /* 头部顶栏样式 */
    .header-top {
        display: flex;
        justify-content: space-between;
        padding: 12px 12px 12px 0; /* 调整右侧内边距 */
        align-items: center;

        /* 搜索包装器 */
        .search-wrapper {
            position: relative;
            margin: 0 12px 0 auto; /* 使用 margin-left: auto 推到右侧并保持间距 */
            flex-grow: 1;
            max-width: 300px;

            /* 搜索图标 */
            .search-icon {
                position: absolute;
                top: 0;
                left: 10px;
                cursor: pointer;
                height: 38px;
                width: 15px;
                display: flex;
                align-items: center;
                font-size: 14px;
                opacity: 0.6;
            }

            /* 搜索输入框 */
            .search-input {
                padding-left: 30px;
                background: rgba(
                    255,
                    255,
                    255,
                    0.2
                ) !important; /* 半透明背景 */
                backdrop-filter: blur(5px); /* 背景模糊效果 */
                -webkit-backdrop-filter: blur(5px); /* Safari浏览器兼容 */
                border: 1px solid rgba(255, 255, 255, 0.3) !important; /* 半透明边框 */
                color: inherit !important; /* 继承文字颜色 */
                border-radius: 16px; /* 圆角边框 */

                .dark & {
                    background: rgba(13, 17, 23, 0.4) !important;
                    border: 1px solid rgba(255, 255, 255, 0.1) !important;
                }
            }
        }
    }

    /* 头部过滤器区域样式 */
    .header-filter {
        margin: 0 12px;
    }
}

/* 列表容器毛玻璃效果 */
.shadow-box {
    background: $glass-bg-light !important; /* 半透明背景 */
    backdrop-filter: blur($glass-blur); /* 背景模糊效果 */
    -webkit-backdrop-filter: blur($glass-blur); /* Safari浏览器兼容 */
    border: 1px solid $glass-border-light !important; /* 半透明边框 */
    border-radius: 16px; /* 圆角边框 */
    box-shadow: $glass-shadow; /* 轻微阴影效果 */

    .dark & {
        background: $glass-bg-dark !important;
        border: 1px solid $glass-border-dark !important;
    }
}

/* 监控列表样式 */
.monitor-list {
    overflow-y: auto;

    &.scrollbar {
        scrollbar-width: thin;
    }
}

/* 选择控制区域样式 */
.selection-controls {
    display: flex;
    gap: 10px;
    align-items: center;
    border-top: 1px solid rgba(0, 0, 0, 0.1);
    padding: 10px;
    background: rgba(255, 255, 255, 0.1); /* 轻微背景色 */
    backdrop-filter: blur(3px);
    -webkit-backdrop-filter: blur(3px);

    .dark & {
        border-top: 1px solid rgba(255, 255, 255, 0.1);
        background: rgba(0, 0, 0, 0.1);
    }

    /* 选择按钮样式 */
    .btn-outline-normal {
        padding: 4px 8px;
        border-radius: 8px;
        cursor: pointer;
        background: rgba(255, 255, 255, 0.2);
        backdrop-filter: blur(5px);
        border: 1px solid rgba(255, 255, 255, 0.3);
        margin-left: 2px;
        transition: all 0.2s ease;
        font-size: 0.9rem;

        .dark & {
            background: rgba(13, 17, 23, 0.4);
            border: 1px solid rgba(255, 255, 255, 0.1);
        }

        &:hover {
            background: rgba(92, 221, 139, 0.2);
            transform: translateY(-1px);
            box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
        }
    }

    /* 选择输入框样式 */
    .select-input {
        margin-right: 5px;
        width: 18px;
        height: 18px;
        cursor: pointer;
    }
}
</style>
