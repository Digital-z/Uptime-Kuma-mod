<template>
    <div>
        <h4 class="mt-5 mb-3">{{ $t("Tags") }}</h4>
        <div v-if="selectedTags.length > 0" class="mb-2 p-1">
            <tag
                v-for="item in selectedTags"
                :key="item.id"
                :item="item"
                :remove="deleteTag"
            />
        </div>
        <div class="p-1 tag-add-container">
            <button
                ref="addTagButton"
                type="button"
                class="btn btn-outline-secondary btn-add"
                :disabled="processing"
                data-testid="add-tag-button"
                @click.stop="showAddDialog"
            >
                <font-awesome-icon class="me-1" icon="plus" /> {{ $t("Add") }}
            </button>

            <!-- 使用Vue的v-if控制显示隐藏，定位在按钮附近 -->
            <div
                v-if="showDialog"
                class="tag-popup-dialog"
                :style="dialogPosition"
                @click.self="closeDialog"
            >
                <div class="tag-dialog-content">
                    <div class="tag-dialog-header">
                        <h5>{{ $t("Add Tag") }}</h5>
                        <button
                            type="button"
                            class="btn-close"
                            @click="closeDialog"
                        ></button>
                    </div>
                    <div class="tag-dialog-body">
                        <vue-multiselect
                            v-model="newDraftTag.select"
                            class="mb-2"
                            :options="tagOptions"
                            :multiple="false"
                            :searchable="true"
                            :placeholder="$t('Add New below or Select...')"
                            track-by="id"
                            label="name"
                            select-label=""
                            deselect-label=""
                        >
                            <template #option="{ option }">
                                <div
                                    class="mx-2 py-1 px-3 rounded d-inline-flex"
                                    style="
                                        margin-top: -5px;
                                        margin-bottom: -5px;
                                        height: 24px;
                                    "
                                    :style="{
                                        color: textColor(option),
                                        backgroundColor:
                                            option.color + ' !important',
                                    }"
                                >
                                    <span> {{ option.name }}</span>
                                </div>
                            </template>
                            <template #singleLabel="{ option }">
                                <div
                                    class="py-1 px-3 rounded d-inline-flex"
                                    style="height: 24px"
                                    :style="{
                                        color: textColor(option),
                                        backgroundColor:
                                            option.color + ' !important',
                                    }"
                                >
                                    <span>{{ option.name }}</span>
                                </div>
                            </template>
                        </vue-multiselect>
                        <div
                            v-if="newDraftTag.select?.name == null"
                            class="d-flex mb-2"
                        >
                            <div class="w-50 pe-2">
                                <input
                                    v-model="newDraftTag.name"
                                    class="form-control"
                                    :class="{
                                        'is-invalid':
                                            validateDraftTag.nameInvalid,
                                    }"
                                    :placeholder="$t('Name')"
                                    data-testid="tag-name-input"
                                    @keydown.enter.prevent="onEnter"
                                />
                                <div class="invalid-feedback">
                                    {{
                                        $t("Tag with this name already exist.")
                                    }}
                                </div>
                            </div>
                            <div class="w-50 ps-2">
                                <vue-multiselect
                                    v-model="newDraftTag.color"
                                    :options="colorOptions"
                                    :multiple="false"
                                    :searchable="true"
                                    :placeholder="$t('color')"
                                    track-by="color"
                                    label="name"
                                    select-label=""
                                    deselect-label=""
                                    data-testid="tag-color-select"
                                >
                                    <template #option="{ option }">
                                        <div
                                            class="mx-2 py-1 px-3 rounded d-inline-flex"
                                            style="height: 24px; color: white"
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
                                            style="height: 24px; color: white"
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
                        </div>
                        <div class="mb-2">
                            <input
                                v-model="newDraftTag.value"
                                class="form-control"
                                :class="{
                                    'is-invalid': validateDraftTag.valueInvalid,
                                }"
                                :placeholder="$t('value (optional)')"
                                data-testid="tag-value-input"
                                @keydown.enter.prevent="onEnter"
                            />
                            <div class="invalid-feedback">
                                {{ $t("Tag with this value already exist.") }}
                            </div>
                        </div>
                        <div class="mb-2">
                            <button
                                type="button"
                                class="btn btn-secondary float-end"
                                :disabled="
                                    processing || validateDraftTag.invalid
                                "
                                data-testid="tag-submit-button"
                                @click.stop="addDraftTag"
                            >
                                {{ $t("Add") }}
                            </button>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
import VueMultiselect from "vue-multiselect";
import { colorOptions } from "../util-frontend";
import Tag from "../components/Tag.vue";

/**
 * @typedef Tag
 * @type {object}
 * @property {number | undefined} id ID of tag assignment
 * @property {number | undefined} monitor_id ID of monitor tag is
 * assigned to
 * @property {number | undefined} tag_id ID of tag
 * @property {string} value Value given to tag
 * @property {string} name Name of tag
 * @property {string} color Colour of tag
 * @property {boolean | undefined} new Should a new tag be created?
 */

export default {
    components: {
        Tag,
        VueMultiselect,
    },
    props: {
        /**
         * Array of tags to be pre-selected
         * @type {Tag[]}
         */
        preSelectedTags: {
            type: Array,
            default: () => [],
        },
    },
    data() {
        return {
            /** @type {boolean} */
            showDialog: false,
            /** @type {Tag[]} */
            existingTags: [],
            processing: false,
            /** @type {Tag[]} */
            newTags: [],
            /** @type {Tag[]} */
            deleteTags: [],
            newDraftTag: {
                name: null,
                select: null,
                color: null,
                value: "",
                invalid: true,
                nameInvalid: false,
            },
        };
    },
    computed: {
        // 计算对话框的位置，使其显示在添加按钮附近
        dialogPosition() {
            // 默认位置，如果按钮引用不可用
            return {
                position: "absolute",
                top: "100%",
                left: "0",
                zIndex: "1050",
            };
        },
        tagOptions() {
            const tagOptions = this.existingTags;
            for (const tag of this.newTags) {
                if (
                    !tagOptions.find(
                        (t) => t.name === tag.name && t.color === tag.color
                    )
                ) {
                    tagOptions.push(tag);
                }
            }
            return tagOptions;
        },
        selectedTags() {
            return this.preSelectedTags
                .concat(this.newTags)
                .filter(
                    (tag) =>
                        !this.deleteTags.find(
                            (monitorTag) => monitorTag.tag_id === tag.tag_id
                        )
                );
        },
        colorOptions() {
            return colorOptions(this);
        },
        validateDraftTag() {
            let nameInvalid = false;
            let valueInvalid = false;
            let invalid = true;
            if (
                this.deleteTags.find(
                    (tag) =>
                        tag.name === this.newDraftTag.select?.name &&
                        tag.value === this.newDraftTag.value
                )
            ) {
                // Undo removing a Tag
                nameInvalid = false;
                valueInvalid = false;
                invalid = false;
            } else if (
                this.existingTags.filter(
                    (tag) => tag.name === this.newDraftTag.name
                ).length > 0 &&
                this.newDraftTag.select == null
            ) {
                // Try to create new tag with existing name
                nameInvalid = true;
                invalid = true;
            } else if (
                this.newTags
                    .concat(this.preSelectedTags)
                    .filter(
                        (tag) =>
                            (tag.name === this.newDraftTag.select?.name &&
                                tag.value === this.newDraftTag.value) ||
                            (tag.name === this.newDraftTag.name &&
                                tag.value === this.newDraftTag.value)
                    ).length > 0
            ) {
                // Try to add a tag with existing name and value
                valueInvalid = true;
                invalid = true;
            } else if (this.newDraftTag.select != null) {
                // Select an existing tag, no need to validate
                invalid = false;
                valueInvalid = false;
            } else if (
                this.newDraftTag.color == null ||
                this.newDraftTag.name === ""
            ) {
                // Missing form inputs
                nameInvalid = false;
                invalid = true;
            } else {
                // Looks valid
                invalid = false;
                nameInvalid = false;
                valueInvalid = false;
            }
            return {
                invalid,
                nameInvalid,
                valueInvalid,
            };
        },
    },
    mounted() {
        this.getExistingTags();
    },
    methods: {
        /**
         * Show the add tag dialog
         * @returns {void}
         */
        showAddDialog() {
            // 在显示对话框前重置状态，防止闪烁
            this.newDraftTag = {
                name: null,
                select: null,
                color: null,
                value: "",
                invalid: true,
                nameInvalid: false,
            };

            // 直接设置显示标志
            this.showDialog = true;

            // 使用nextTick确保DOM已更新
            this.$nextTick(() => {
                // 如果需要，可以在这里添加额外的定位逻辑
                // 例如，如果对话框超出视口边界，可以调整位置
            });
        },

        /**
         * 关闭对话框
         * @returns {void}
         */
        closeDialog() {
            this.showDialog = false;
        },

        /**
         * Get all existing tags
         * @returns {void}
         */
        getExistingTags() {
            this.$root.getSocket().emit("getTags", (res) => {
                if (res.ok) {
                    this.existingTags = res.tags;
                } else {
                    this.$root.toastError(res.msg);
                }
            });
        },
        /**
         * Delete the specified tag
         * @param {object} item Object representing tag to delete
         * @returns {void}
         */
        deleteTag(item) {
            if (item.new) {
                // Undo Adding a new Tag
                this.newTags = this.newTags.filter(
                    (tag) =>
                        !(tag.name === item.name && tag.value === item.value)
                );
            } else {
                // Remove an Existing Tag
                this.deleteTags.push(item);
            }
        },
        /**
         * Get colour of text inside the tag
         * @param {object} option The tag that needs to be displayed.
         * Defaults to "white" unless the tag has no color, which will
         * then return the body color (based on application theme)
         * @returns {string} Text color
         */
        textColor(option) {
            if (option.color) {
                return "white";
            } else {
                return this.$root.theme === "light"
                    ? "var(--bs-body-color)"
                    : "inherit";
            }
        },
        /**
         * Add a draft tag
         * @returns {void}
         */
        addDraftTag() {
            console.log("Adding Draft Tag: ", this.newDraftTag);
            if (this.newDraftTag.select != null) {
                if (
                    this.deleteTags.find(
                        (tag) =>
                            tag.name === this.newDraftTag.select.name &&
                            tag.value === this.newDraftTag.value
                    )
                ) {
                    // Undo removing a tag
                    this.deleteTags = this.deleteTags.filter(
                        (tag) =>
                            !(
                                tag.name === this.newDraftTag.select.name &&
                                tag.value === this.newDraftTag.value
                            )
                    );
                } else {
                    // Add an existing Tag
                    this.newTags.push({
                        id: this.newDraftTag.select.id,
                        color: this.newDraftTag.select.color,
                        name: this.newDraftTag.select.name,
                        value: this.newDraftTag.value,
                        new: true,
                    });
                }
            } else {
                // Add new Tag
                this.newTags.push({
                    color: this.newDraftTag.color.color,
                    name: this.newDraftTag.name.trim(),
                    value: this.newDraftTag.value,
                    new: true,
                });
            }
            this.clearDraftTag();
        },
        /**
         * Remove a draft tag
         * @returns {void}
         */
        clearDraftTag() {
            this.newDraftTag = {
                name: null,
                select: null,
                color: null,
                value: "",
                invalid: true,
                nameInvalid: false,
            };
            this.closeDialog();
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
        /**
         * Handle pressing Enter key when inside the modal
         * @returns {void}
         */
        onEnter() {
            if (!this.validateDraftTag.invalid) {
                this.addDraftTag();
            }
        },
        /**
         * Submit the form data
         * @param {number} monitorId ID of monitor this change affects
         * @returns {Promise<void>}
         */
        async submit(monitorId) {
            console.log(`Submitting tag changes for monitor ${monitorId}...`);
            this.processing = true;

            for (const newTag of this.newTags) {
                let tagId;
                if (newTag.id == null) {
                    // Create a New Tag
                    let newTagResult;
                    await this.addTagAsync(newTag).then((res) => {
                        if (!res.ok) {
                            this.$root.toastError(res.msg);
                            newTagResult = false;
                        }
                        newTagResult = res.tag;
                    });
                    if (!newTagResult) {
                        // abort
                        this.processing = false;
                        return;
                    }
                    tagId = newTagResult.id;
                    // Assign the new ID to the tags of the same name & color
                    this.newTags.map((tag) => {
                        if (
                            tag.name === newTag.name &&
                            tag.color === newTag.color
                        ) {
                            tag.id = newTagResult.id;
                        }
                    });
                } else {
                    tagId = newTag.id;
                }

                let newMonitorTagResult;
                // Assign tag to monitor
                await this.addMonitorTagAsync(
                    tagId,
                    monitorId,
                    newTag.value
                ).then((res) => {
                    if (!res.ok) {
                        this.$root.toastError(res.msg);
                        newMonitorTagResult = false;
                    }
                    newMonitorTagResult = true;
                });
                if (!newMonitorTagResult) {
                    // abort
                    this.processing = false;
                    return;
                }
            }

            for (const deleteTag of this.deleteTags) {
                let deleteMonitorTagResult;
                await this.deleteMonitorTagAsync(
                    deleteTag.tag_id,
                    deleteTag.monitor_id,
                    deleteTag.value
                ).then((res) => {
                    if (!res.ok) {
                        this.$root.toastError(res.msg);
                        deleteMonitorTagResult = false;
                    }
                    deleteMonitorTagResult = true;
                });
                if (!deleteMonitorTagResult) {
                    // abort
                    this.processing = false;
                    return;
                }
            }

            this.getExistingTags();
            this.newTags = [];
            this.deleteTags = [];
            this.processing = false;
        },
    },
};
</script>

<style lang="scss" scoped>
.btn-add {
    width: 100%;
    border: 2px solid rgba(0, 0, 0, 0.2) !important; /* 更明显的边框 */
    background-color: rgba(255, 255, 255, 0.2) !important;

    /* 暗色模式样式 */
    .dark & {
        border: 2px solid rgba(255, 255, 255, 0.2) !important; /* 暗色模式下更明显的边框 */
        background-color: rgba(13, 17, 23, 0.4) !important;
    }

    &:hover {
        border-color: rgba(
            92,
            221,
            139,
            0.5
        ) !important; /* 悬停时边框颜色变化 */
    }
}

/* 标签添加容器，用于定位弹出框 */
.tag-add-container {
    position: relative;
}

/* 标签弹出对话框样式 */
.tag-popup-dialog {
    position: absolute;
    top: 100%;
    left: 0;
    width: 100%;
    max-width: 400px;
    z-index: 1050;
    margin-top: 5px;
}

.tag-dialog-content {
    position: relative;
    display: flex;
    flex-direction: column;
    width: 100%;
    /* 使用透明背景效果 */
    background-color: rgba(255, 255, 255, 0.2); /* 更透明的背景 */
    backdrop-filter: blur(5px); /* 减少模糊效果 */
    border: 1px solid var(--glass-border);
    border-radius: 0.5rem;
    box-shadow: 0 0.25rem 0.5rem rgba(0, 0, 0, 0.1); /* 减轻阴影 */

    /* 暗色模式样式 */
    .dark & {
        background-color: rgba(22, 27, 34, 0.2); /* 更透明的背景 */
        box-shadow: 0 0.25rem 0.5rem rgba(0, 0, 0, 0.15);
    }
}

.tag-dialog-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 0.75rem 1rem;
    border-bottom: 1px solid var(--glass-border);
    /* 添加透明效果 */
    background-color: rgba(var(--bs-primary-rgb), 0.05); /* 更透明的背景 */
    backdrop-filter: blur(5px); /* 减少模糊效果 */
    border-top-left-radius: 0.5rem;
    border-top-right-radius: 0.5rem;

    /* 暗色模式样式 */
    .dark & {
        background-color: rgba(var(--bs-primary-rgb), 0.08); /* 更透明的背景 */
    }

    h5 {
        margin: 0;
        font-size: 1.1rem;
        font-weight: 500;
        color: var(--bs-body-color);
        /* 移除背景和边框，只保留文字 */
        background-color: transparent;
        padding: 0.25rem 0;
        border: none;

        /* 暗色模式样式 */
        .dark & {
            color: var(--bs-body-color);
        }
    }
}

.tag-dialog-body {
    padding: 1rem;
}

/* 确保vue-multiselect组件在自定义对话框中正常显示 */
.tag-dialog-body .multiselect {
    background-color: transparent !important;
}

/* 确保下拉菜单样式与主题一致 */
.tag-dialog-body .multiselect__content-wrapper {
    /* 使用与主题一致的背景色 */
    background: var(--glass-bg) !important;
    border: 2px solid rgba(0, 0, 0, 0.2) !important; /* 更明显的边框 */
    backdrop-filter: blur(10px);

    /* 暗色模式样式 */
    .dark & {
        border: 2px solid rgba(255, 255, 255, 0.2) !important; /* 暗色模式下更明显的边框 */
    }
}

/* 标签样式优化 */
.tag-dialog-body .multiselect__tag {
    background-color: rgba(var(--bs-primary-rgb), 0.2) !important;
    color: var(--bs-body-color) !important;
    border: 1px solid rgba(var(--bs-primary-rgb), 0.3) !important;
    backdrop-filter: blur(5px) !important;
    margin-right: 5px !important;
    padding: 4px 26px 4px 10px !important;
    border-radius: 4px !important;

    /* 暗色模式样式 */
    .dark & {
        background-color: rgba(var(--bs-primary-rgb), 0.3) !important;
        border-color: rgba(var(--bs-primary-rgb), 0.4) !important;
    }
}

/* 标签删除按钮样式 */
.tag-dialog-body .multiselect__tag-icon {
    background-color: transparent !important;
    color: var(--bs-body-color) !important;
    opacity: 0.7 !important;

    &:hover {
        background-color: rgba(var(--bs-danger-rgb), 0.2) !important;
        color: var(--bs-danger) !important;
        opacity: 1 !important;
    }
}

/* 定义CSS变量，根据主题切换 */
:root {
    --glass-bg: rgba(255, 255, 255, 0.2); /* 更透明的背景 */
    --glass-border: rgba(0, 0, 0, 0.08); /* 更透明的边框 */
}

.dark {
    --glass-bg: rgba(22, 27, 34, 0.2); /* 更透明的背景 */
    --glass-border: rgba(255, 255, 255, 0.08); /* 更透明的边框 */
}
</style>
