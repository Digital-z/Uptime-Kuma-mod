<template>
    <teleport to="body">
        <!-- 使用v-if而不是v-show，确保组件完全卸载和重新挂载 -->
        <div v-if="visible" class="custom-modal-container" @click.self="hide">
            <form @submit.prevent="submit">
                <div
                    class="custom-modal"
                    :class="{ 'custom-modal-show': animationState === 'show' }"
                >
                    <div class="custom-modal-header">
                        <h5 class="custom-modal-title">
                            {{ $t("Edit Tag") }}
                        </h5>
                        <button
                            type="button"
                            class="custom-close-button"
                            aria-label="Close"
                            @click="hide"
                        >
                            ×
                        </button>
                    </div>
                    <div class="custom-modal-body">
                        <div class="mb-3">
                            <label for="tag-name" class="form-label">{{
                                $t("Name")
                            }}</label>
                            <input
                                id="tag-name"
                                v-model="tag.name"
                                type="text"
                                class="form-control"
                                :class="{ 'is-invalid': nameInvalid }"
                                required
                            />
                            <div class="invalid-feedback">
                                {{ $t("Tag with this name already exist.") }}
                            </div>
                        </div>

                        <div class="mb-3">
                            <label for="tag-color" class="form-label">{{
                                $t("color")
                            }}</label>
                            <div class="d-flex">
                                <div class="col-8 pe-1">
                                    <vue-multiselect
                                        v-model="selectedColor"
                                        :options="colorOptions"
                                        :multiple="false"
                                        :searchable="true"
                                        :placeholder="$t('color')"
                                        track-by="color"
                                        label="name"
                                        select-label=""
                                        deselect-label=""
                                    >
                                        <template #option="{ option }">
                                            <div
                                                class="mx-2 py-1 px-3 rounded d-inline-flex"
                                                style="
                                                    height: 24px;
                                                    color: white;
                                                "
                                                :style="{
                                                    backgroundColor:
                                                        option.color +
                                                        ' !important',
                                                }"
                                            >
                                                <span>{{ option.name }}</span>
                                            </div>
                                        </template>
                                        <template #singleLabel="{ option }">
                                            <div
                                                class="py-1 px-3 rounded d-inline-flex"
                                                style="
                                                    height: 24px;
                                                    color: white;
                                                "
                                                :style="{
                                                    backgroundColor:
                                                        option.color +
                                                        ' !important',
                                                }"
                                            >
                                                <span>{{ option.name }}</span>
                                            </div>
                                        </template>
                                    </vue-multiselect>
                                </div>
                                <div class="col-4 ps-1">
                                    <input
                                        id="tag-color-hex"
                                        v-model="tag.color"
                                        type="text"
                                        class="form-control"
                                    />
                                </div>
                            </div>
                        </div>

                        <div class="mb-3">
                            <label for="tag-monitors" class="form-label">{{
                                $tc("Monitor", selectedMonitors.length)
                            }}</label>
                            <div class="tag-monitors-list">
                                <router-link
                                    v-for="monitor in selectedMonitors"
                                    :key="monitor.id"
                                    class="d-flex align-items-center justify-content-between text-decoration-none tag-monitors-list-row py-2 px-3"
                                    :to="monitorURL(monitor.id)"
                                    @click="hide"
                                >
                                    <span>{{ monitor.name }}</span>
                                    <button
                                        type="button"
                                        class="btn-rm-monitor btn btn-outline-danger ms-2 py-1"
                                        @click.stop.prevent="
                                            removeMonitor(monitor.id)
                                        "
                                    >
                                        <font-awesome-icon
                                            class=""
                                            icon="times"
                                        />
                                    </button>
                                </router-link>
                            </div>
                            <div v-if="allMonitorList.length > 0" class="pt-3">
                                <label class="form-label"
                                    >{{ $t("Add a monitor") }}:</label
                                >
                                <VueMultiselect
                                    v-model="selectedAddMonitor"
                                    :options="allMonitorList"
                                    :multiple="false"
                                    :searchable="true"
                                    :placeholder="$t('Add a monitor')"
                                    label="name"
                                    trackBy="name"
                                    class="mt-1"
                                >
                                    <template #option="{ option }">
                                        <div class="d-inline-flex">
                                            <span
                                                >{{ option.name }}
                                                <Tag
                                                    v-for="monitorTag in option.tags"
                                                    :key="monitorTag"
                                                    :item="monitorTag"
                                                    :size="'sm'"
                                            /></span>
                                        </div>
                                    </template>
                                </VueMultiselect>
                            </div>
                        </div>
                    </div>

                    <div class="custom-modal-footer">
                        <button
                            v-if="tag && tag.id !== null"
                            type="button"
                            class="btn btn-danger"
                            :disabled="processing"
                            @click="deleteConfirm"
                        >
                            {{ $t("Delete") }}
                        </button>
                        <button
                            type="submit"
                            class="btn btn-primary"
                            :disabled="processing"
                        >
                            <div
                                v-if="processing"
                                class="spinner-border spinner-border-sm me-1"
                            ></div>
                            {{ $t("Save") }}
                        </button>
                    </div>
                </div>
            </form>
        </div>
    </teleport>

    <Confirm
        ref="confirmDelete"
        btn-style="btn-danger"
        :yes-text="$t('Yes')"
        :no-text="$t('No')"
        @yes="deleteTag"
    >
        {{ $t("confirmDeleteTagMsg") }}
    </Confirm>
</template>

<script>
import Confirm from "./Confirm.vue";
import Tag from "./Tag.vue";
import VueMultiselect from "vue-multiselect";
import { colorOptions } from "../util-frontend";
import { getMonitorRelativeURL } from "../util.ts";

export default {
    components: {
        VueMultiselect,
        Confirm,
        Tag,
    },
    props: {
        updated: {
            type: Function,
            default: () => {},
        },
        existingTags: {
            type: Array,
            default: () => [],
        },
    },
    data() {
        return {
            // 自定义模态框状态
            visible: false,
            animationState: "hidden",
            animationTimeout: null,
            lockShow: false, // 防止重复显示

            processing: false,
            selectedColor: {
                name: null,
                color: null,
            },
            tag: {
                id: null,
                name: "",
                color: "",
                // Do not set default value here, please scroll to show()
            },
            monitors: [],
            removingMonitor: [],
            addingMonitor: [],
            selectedAddMonitor: null,
            nameInvalid: false,
        };
    },

    computed: {
        colorOptions() {
            if (
                !colorOptions(this).find(
                    (option) => option.color === this.tag.color
                )
            ) {
                return colorOptions(this).concat({
                    name: "custom",
                    color: this.tag.color,
                });
            } else {
                return colorOptions(this);
            }
        },
        selectedMonitors() {
            return this.monitors
                .concat(
                    Object.values(this.$root.monitorList).filter((monitor) =>
                        this.addingMonitor.includes(monitor.id)
                    )
                )
                .filter(
                    (monitor) => !this.removingMonitor.includes(monitor.id)
                );
        },
        allMonitorList() {
            return Object.values(this.$root.monitorList).filter(
                (monitor) => !this.selectedMonitors.includes(monitor)
            );
        },
    },

    watch: {
        // Set color option to "Custom" when a unknown color is entered
        "tag.color"(to, from) {
            if (
                to !== "" &&
                colorOptions(this).find((x) => x.color === to) == null
            ) {
                this.selectedColor.name = this.$t("Custom");
                this.selectedColor.color = to;
            }
        },
        "tag.name"(to, from) {
            if (to != null) {
                this.validate();
            }
        },
        selectedColor(to, from) {
            if (to != null) {
                this.tag.color = to.color;
            }
        },
        /**
         * Selected a monitor and add to the list.
         * @param {object} monitor Monitor to add
         * @returns {void}
         */
        selectedAddMonitor(monitor) {
            if (monitor) {
                if (this.removingMonitor.includes(monitor.id)) {
                    this.removingMonitor = this.removingMonitor.filter(
                        (id) => id !== monitor.id
                    );
                } else {
                    this.addingMonitor.push(monitor.id);
                }
                this.selectedAddMonitor = null;
            }
        },
    },

    mounted() {
        // 不再需要初始化Bootstrap Modal
    },

    beforeUnmount() {
        // 清除可能存在的超时
        this.clearTimeouts();
        // 确保移除事件监听器
        document.removeEventListener("keydown", this.handleEscKey);
    },

    methods: {
        /**
         * 清除所有超时
         * @returns {void}
         */
        clearTimeouts() {
            if (this.animationTimeout) {
                clearTimeout(this.animationTimeout);
                this.animationTimeout = null;
            }
        },

        /**
         * 处理ESC键按下
         * @param {KeyboardEvent} event 键盘事件
         * @returns {void}
         */
        handleEscKey(event) {
            if (event.key === "Escape") {
                this.hide();
            }
        },

        /**
         * Show confirmation for deleting a tag
         * @returns {void}
         */
        deleteConfirm() {
            this.$refs.confirmDelete.show();
        },

        /**
         * Reset the editTag form
         * @returns {void}
         */
        reset() {
            this.selectedColor = null;
            this.tag = {
                id: null,
                name: "",
                color: "",
            };
            this.monitors = [];
            this.removingMonitor = [];
            this.addingMonitor = [];
        },

        /**
         * Check for existing tags of the same name, set invalid input
         * @returns {boolean} True if editing tag is valid
         */
        validate() {
            this.nameInvalid = false;
            const sameName = this.existingTags.find(
                (existingTag) => existingTag.name === this.tag.name
            );
            if (sameName != null && sameName.id !== this.tag.id) {
                this.nameInvalid = true;
                return false;
            }
            return true;
        },

        /**
         * 显示对话框
         * @param {object} tag tag object to edit
         * @returns {void}
         */
        show(tag) {
            // 如果已经显示或者锁定中，不做任何操作
            if (this.visible || this.lockShow) {
                return;
            }

            // 锁定显示操作，防止重复调用
            this.lockShow = true;

            // 加载标签数据
            if (tag) {
                this.selectedColor = this.colorOptions.find(
                    (x) => x.color === tag.color
                ) ?? {
                    name: this.$t("Custom"),
                    color: tag.color,
                };
                this.tag.id = tag.id;
                this.tag.name = tag.name;
                this.tag.color = tag.color;
                this.monitors = this.monitorsByTag(tag.id);
                this.removingMonitor = [];
                this.addingMonitor = [];
                this.selectedAddMonitor = null;
            }

            // 先显示背景
            this.visible = true;

            // 添加ESC键监听
            document.addEventListener("keydown", this.handleEscKey);

            // 使用nextTick确保DOM已更新
            this.$nextTick(() => {
                // 添加一个小延迟，确保CSS过渡效果正常工作
                this.animationTimeout = setTimeout(() => {
                    this.animationState = "show";
                    // 防止背景滚动
                    document.body.style.overflow = "hidden";
                    // 解除锁定
                    this.lockShow = false;
                }, 50);
            });
        },

        /**
         * 隐藏对话框
         * @returns {void}
         */
        hide() {
            // 如果已经隐藏，不做任何操作
            if (!this.visible) {
                return;
            }

            // 先触发隐藏动画
            this.animationState = "hidden";

            // 移除ESC键监听
            document.removeEventListener("keydown", this.handleEscKey);

            // 等待动画完成后再移除元素
            this.animationTimeout = setTimeout(() => {
                this.visible = false;
                // 恢复背景滚动
                document.body.style.overflow = "";
            }, 300); // 300ms是过渡动画的时间
        },

        /**
         * Submit tag and monitorTag changes to server
         * @returns {Promise<void>}
         */
        async submit() {
            this.processing = true;
            let editResult = true;

            if (!this.validate()) {
                this.processing = false;
                return;
            }

            if (this.tag.id == null) {
                await this.addTagAsync(this.tag).then((res) => {
                    if (!res.ok) {
                        this.$root.toastRes(res.msg);
                        editResult = false;
                    } else {
                        this.tag.id = res.tag.id;
                        this.updated();
                    }
                });
            }

            if (!editResult) {
                return;
            }

            for (let addId of this.addingMonitor) {
                await this.addMonitorTagAsync(this.tag.id, addId, "").then(
                    (res) => {
                        if (!res.ok) {
                            this.$root.toastError(res.msg);
                            editResult = false;
                        }
                    }
                );
            }

            for (let removeId of this.removingMonitor) {
                this.monitors
                    .find((monitor) => monitor.id === removeId)
                    ?.tags.forEach(async (monitorTag) => {
                        await this.deleteMonitorTagAsync(
                            this.tag.id,
                            removeId,
                            monitorTag.value
                        ).then((res) => {
                            if (!res.ok) {
                                this.$root.toastError(res.msg);
                                editResult = false;
                            }
                        });
                    });
            }

            this.$root.getSocket().emit("editTag", this.tag, (res) => {
                this.$root.toastRes(res);
                this.processing = false;

                if (res.ok && editResult) {
                    this.updated();
                    this.hide();
                }
            });
        },

        /**
         * Delete the editing tag from server
         * @returns {Promise<void>}
         */
        async deleteTag() {
            this.processing = true;
            await this.deleteTagAsync(this.tag.id).then((res) => {
                this.$root.toastRes(res);
                this.processing = false;

                if (res.ok) {
                    this.updated();
                    this.hide();
                }
            });
        },

        /**
         * Remove a monitor from the monitors list locally
         * @param {number} id id of the tag to remove
         * @returns {void}
         */
        removeMonitor(id) {
            if (this.addingMonitor.includes(id)) {
                this.addingMonitor = this.addingMonitor.filter((x) => x !== id);
            } else {
                this.removingMonitor.push(id);
            }
        },

        /**
         * Get monitors which has a specific tag locally
         * @param {number} tagId id of the tag to filter
         * @returns {object[]} list of monitors which has a specific tag
         */
        monitorsByTag(tagId) {
            return Object.values(this.$root.monitorList).filter((monitor) => {
                return monitor.tags.find(
                    (monitorTag) => monitorTag.tag_id === tagId
                );
            });
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
         * Add a tag asynchronously
         * @param {object} newTag Object representing new tag to add
         * @returns {Promise<void>}
         */
        addTagAsync(newTag) {
            return new Promise((resolve) => {
                this.$root.getSocket().emit("addTag", newTag, resolve);
            });
        },

        /**
         * Delete a tag asynchronously
         * @param {number} tagId ID of tag to delete
         * @returns {Promise<void>}
         */
        deleteTagAsync(tagId) {
            return new Promise((resolve) => {
                this.$root.getSocket().emit("deleteTag", tagId, resolve);
            });
        },

        /**
         * Add a tag to a monitor asynchronously
         * @param {number} tagId ID of tag to add
         * @param {number} monitorId ID of monitor to add tag to
         * @param {string} value Value of tag
         * @returns {Promise<void>}
         */
        addMonitorTagAsync(tagId, monitorId, value) {
            return new Promise((resolve) => {
                this.$root
                    .getSocket()
                    .emit("addMonitorTag", tagId, monitorId, value, resolve);
            });
        },
        /**
         * Delete a tag from a monitor asynchronously
         * @param {number} tagId ID of tag to remove
         * @param {number} monitorId ID of monitor to remove tag from
         * @param {string} value Value of tag
         * @returns {Promise<void>}
         */
        deleteMonitorTagAsync(tagId, monitorId, value) {
            return new Promise((resolve) => {
                this.$root
                    .getSocket()
                    .emit("deleteMonitorTag", tagId, monitorId, value, resolve);
            });
        },
    },
};
</script>

<style lang="scss" scoped>
@import "../assets/vars.scss";

/* 模态框容器 - 全屏覆盖 */
.custom-modal-container {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.5);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1050;
    opacity: 1;
    transition: opacity 0.3s ease;
    /* 确保模态框容器覆盖整个视口 */
    margin: 0;
    padding: 0;
}

/* 模态框本身 */
.custom-modal {
    background-color: #fff;
    border-radius: 0.3rem;
    box-shadow: 0 0.5rem 1rem rgba(0, 0, 0, 0.15);
    width: 500px;
    max-width: 90%;
    opacity: 0;
    transform: translateY(-20px);
    transition: all 0.3s ease;
    margin: auto; /* 确保水平居中 */
    position: relative; /* 确保z-index生效 */
}

/* 显示状态的模态框 */
.custom-modal-show {
    opacity: 1;
    transform: translateY(0);
}

/* 模态框头部 */
.custom-modal-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 1rem;
    border-bottom: 1px solid #dee2e6;
}

/* 模态框标题 */
.custom-modal-title {
    margin: 0;
    font-size: 1.25rem;
}

/* 关闭按钮 */
.custom-close-button {
    background: transparent;
    border: none;
    font-size: 1.5rem;
    font-weight: 700;
    line-height: 1;
    color: #000;
    opacity: 0.5;
    cursor: pointer;
    padding: 0;
    margin: 0;
}

.custom-close-button:hover {
    opacity: 0.75;
}

/* 模态框内容 */
.custom-modal-body {
    padding: 1rem;
    overflow: auto;
    max-height: 80vh;
}

/* 模态框底部 */
.custom-modal-footer {
    display: flex;
    align-items: center;
    justify-content: flex-end;
    padding: 1rem;
    border-top: 1px solid #dee2e6;
    gap: 0.5rem;
}

.btn-rm-monitor {
    padding-left: 11px;
    padding-right: 11px;
}

.tag-monitors-list {
    max-height: 40vh;
    overflow-y: scroll;
}

.tag-monitors-list .tag-monitors-list-row {
    cursor: pointer;
    border-bottom: 1px solid rgba(0, 0, 0, 0.125);

    .dark & {
        border-bottom: 1px solid $dark-border-color;
    }

    &:hover {
        background-color: $highlight-white;
    }

    .dark &:hover {
        background-color: $dark-bg2;
    }
}

/* 暗色主题适配 */
.dark {
    .custom-modal {
        background-color: #343a40;
        color: #f8f9fa;
    }

    .custom-modal-header {
        border-color: #495057;
    }

    .custom-modal-footer {
        border-color: #495057;
    }

    .custom-close-button {
        color: #f8f9fa;
    }

    .form-text,
    p {
        color: $dark-font-color;
    }
}
</style>
