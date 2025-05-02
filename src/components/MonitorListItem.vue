<template>
    <div>
        <div :style="depthMargin" class="monitor-item-container">
            <!-- Checkbox -->
            <div v-if="isSelectMode" class="select-input-wrapper">
                <input
                    class="form-check-input select-input"
                    type="checkbox"
                    :aria-label="$t('Check/Uncheck')"
                    :checked="isSelected(monitor.id)"
                    @click.stop="toggleSelection"
                />
            </div>

            <!-- 监控项主体内容 -->
            <router-link
                :to="monitorURL(monitor.id)"
                class="item"
                :class="{
                    disabled: !monitor.active,
                    'with-checkbox': isSelectMode,
                }"
            >
                <div class="row">
                    <div
                        class="col-6 small-padding"
                        :class="{
                            'monitor-item':
                                $root.userHeartbeatBar == 'bottom' ||
                                $root.userHeartbeatBar == 'none',
                        }"
                    >
                        <div class="info">
                            <Uptime :monitor="monitor" type="24" :pill="true" />
                            <span
                                v-if="hasChildren"
                                class="collapse-padding"
                                @click.prevent="changeCollapsed"
                            >
                                <font-awesome-icon
                                    icon="chevron-down"
                                    class="animated"
                                    :class="{ collapsed: isCollapsed }"
                                />
                            </span>
                            {{ monitor.name }}
                        </div>
                        <div v-if="monitor.tags.length > 0" class="tags gap-1">
                            <Tag
                                v-for="tag in monitor.tags"
                                :key="tag"
                                :item="tag"
                                :size="'sm'"
                            />
                        </div>
                    </div>
                    <div
                        v-show="$root.userHeartbeatBar == 'normal'"
                        :key="$root.userHeartbeatBar"
                        class="col-6"
                    >
                        <HeartbeatBar
                            ref="heartbeatBar"
                            size="small"
                            :monitor-id="monitor.id"
                        />
                    </div>
                </div>

                <div v-if="$root.userHeartbeatBar == 'bottom'" class="row">
                    <div class="col-12 bottom-style">
                        <HeartbeatBar
                            ref="heartbeatBar"
                            size="small"
                            :monitor-id="monitor.id"
                        />
                    </div>
                </div>
            </router-link>
        </div>

        <transition name="slide-fade-up">
            <div v-if="!isCollapsed" class="childs">
                <MonitorListItem
                    v-for="(item, index) in sortedChildMonitorList"
                    :key="index"
                    :monitor="item"
                    :isSelectMode="isSelectMode"
                    :isSelected="isSelected"
                    :select="select"
                    :deselect="deselect"
                    :depth="depth + 1"
                    :filter-func="filterFunc"
                    :sort-func="sortFunc"
                />
            </div>
        </transition>
    </div>
</template>

<script>
import HeartbeatBar from "../components/HeartbeatBar.vue";
import Tag from "../components/Tag.vue";
import Uptime from "../components/Uptime.vue";
import { getMonitorRelativeURL } from "../util.ts";

export default {
    name: "MonitorListItem",
    components: {
        Uptime,
        HeartbeatBar,
        Tag,
    },
    props: {
        /** Monitor this represents */
        monitor: {
            type: Object,
            default: null,
        },
        /** If the user is in select mode */
        isSelectMode: {
            type: Boolean,
            default: false,
        },
        /** How many ancestors are above this monitor */
        depth: {
            type: Number,
            default: 0,
        },
        /** Callback to determine if monitor is selected */
        isSelected: {
            type: Function,
            default: () => {},
        },
        /** Callback fired when monitor is selected */
        select: {
            type: Function,
            default: () => {},
        },
        /** Callback fired when monitor is deselected */
        deselect: {
            type: Function,
            default: () => {},
        },
        /** Function to filter child monitors */
        filterFunc: {
            type: Function,
            default: () => {},
        },
        /** Function to sort child monitors */
        sortFunc: {
            type: Function,
            default: () => {},
        },
    },
    data() {
        return {
            isCollapsed: true,
        };
    },
    computed: {
        sortedChildMonitorList() {
            let result = Object.values(this.$root.monitorList);

            // Get children
            result = result.filter(
                (childMonitor) => childMonitor.parent === this.monitor.id
            );

            // Run filter on children
            result = result.filter(this.filterFunc);

            result.sort(this.sortFunc);

            return result;
        },
        hasChildren() {
            return this.sortedChildMonitorList.length > 0;
        },
        depthMargin() {
            return {
                paddingLeft: `${31 * this.depth}px`,
            };
        },
    },
    watch: {
        isSelectMode() {
            // TODO: Resize the heartbeat bar, but too slow
            // this.$refs.heartbeatBar.resize();
        },
    },
    beforeMount() {
        // Always unfold if monitor is accessed directly
        if (
            this.monitor.childrenIDs.includes(parseInt(this.$route.params.id))
        ) {
            this.isCollapsed = false;
            return;
        }

        // Set collapsed value based on local storage
        let storage = window.localStorage.getItem("monitorCollapsed");
        if (storage === null) {
            return;
        }

        let storageObject = JSON.parse(storage);
        if (storageObject[`monitor_${this.monitor.id}`] == null) {
            return;
        }

        this.isCollapsed = storageObject[`monitor_${this.monitor.id}`];
    },
    methods: {
        /**
         * Changes the collapsed value of the current monitor and saves
         * it to local storage
         * @returns {void}
         */
        changeCollapsed() {
            this.isCollapsed = !this.isCollapsed;

            // Save collapsed value into local storage
            let storage = window.localStorage.getItem("monitorCollapsed");
            let storageObject = {};
            if (storage !== null) {
                storageObject = JSON.parse(storage);
            }
            storageObject[`monitor_${this.monitor.id}`] = this.isCollapsed;

            window.localStorage.setItem(
                "monitorCollapsed",
                JSON.stringify(storageObject)
            );
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
         * Toggle selection of monitor
         * @returns {void}
         */
        toggleSelection() {
            if (this.isSelected(this.monitor.id)) {
                this.deselect(this.monitor.id);
            } else {
                this.select(this.monitor.id);
            }
        },
    },
};
</script>

<style lang="scss" scoped>
@import "../assets/vars.scss";

/* 监控项容器样式 */
.monitor-item-container {
    position: relative;
    display: flex;
    align-items: center;
    margin: 5px 10px;
}

/* 监控项链接样式 */
.item {
    display: block;
    padding: 10px 15px;
    border-radius: 14px;
    text-decoration: none;
    transition: all 0.3s ease;
    color: inherit;
    position: relative;
    overflow: hidden;
    flex-grow: 1;

    /* 毛玻璃效果 */
    background: rgba(255, 255, 255, 0.25) !important; /* 半透明白色背景 */
    backdrop-filter: blur(5px); /* 背景模糊效果 */
    -webkit-backdrop-filter: blur(5px); /* Safari浏览器兼容 */
    border: 1px solid rgba(255, 255, 255, 0.3); /* 半透明边框 */
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05); /* 轻微阴影 */

    /* 选择模式下的样式调整 */
    &.with-checkbox {
        margin-left: 30px; /* 为复选框留出空间 */
    }

    /* 悬停效果 */
    &:hover {
        background: rgba(255, 255, 255, 0.4) !important;
        box-shadow: 0 3px 12px rgba(0, 0, 0, 0.1);
        color: inherit;
    }

    /* 禁用状态 */
    &.disabled {
        opacity: 0.6;
    }

    /* 暗色模式样式 */
    .dark & {
        background: rgba(13, 17, 23, 0.4) !important;
        border: 1px solid rgba(255, 255, 255, 0.1);

        &:hover {
            background: rgba(13, 17, 23, 0.6) !important;
        }
    }
}

/* 监控项信息区域样式 */
.info {
    display: flex;
    align-items: center;
    gap: 5px;
    font-weight: 500;
}

/* 标签容器样式 */
.tags {
    margin-top: 4px;
    padding-left: 67px;
    display: flex;
    flex-wrap: wrap;
    gap: 3px;
}

/* 选择框包装器样式 */
.select-input-wrapper {
    position: absolute;
    left: 0; /* 从容器左侧开始 */
    top: 50%;
    transform: translateY(-50%);
    z-index: 10; /* 提高层级，确保在最上层 */
    width: 30px; /* 增加宽度，确保有足够的点击区域 */
    height: 30px;
    display: flex;
    justify-content: center;
    align-items: center;
    background-color: rgba(255, 255, 255, 0.1); /* 轻微背景色，使复选框更明显 */
    border-radius: 50%; /* 圆形背景 */
}

/* 选择输入框样式 */
.select-input {
    cursor: pointer;
    margin: 0;
    width: 16px;
    height: 16px;
}

/* 折叠按钮内边距 */
.collapse-padding {
    padding: 0 5px;
    cursor: pointer;
}

/* 动画效果 */
.animated {
    transition: transform 0.3s ease;

    &.collapsed {
        transform: rotate(-90deg);
    }
}

/* 滑动淡入动画 */
.slide-fade-up-enter-active {
    transition: all 0.3s ease;
}

.slide-fade-up-leave-active {
    transition: all 0.3s cubic-bezier(1, 0.5, 0.8, 1);
}

.slide-fade-up-enter-from,
.slide-fade-up-leave-to {
    transform: translateY(10px);
    opacity: 0;
}

/* 底部样式 */
.bottom-style {
    padding-left: 67px;
    margin-top: 5px;
}

/* 小内边距 */
.small-padding {
    padding-left: 5px !important;
    padding-right: 5px !important;
}
</style>
