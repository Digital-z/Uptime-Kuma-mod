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
                            {{ $t("Add API Key") }}
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
                        <!-- Name -->
                        <div class="mb-3">
                            <label for="name" class="form-label">{{
                                $t("Name")
                            }}</label>
                            <input
                                id="name"
                                v-model="key.name"
                                type="text"
                                class="form-control"
                                required
                            />
                        </div>

                        <!-- Expiry -->
                        <div class="my-3">
                            <label class="form-label">{{
                                $t("Expiry date")
                            }}</label>
                            <div class="d-flex flex-row align-items-center">
                                <div class="col-6">
                                    <Datepicker
                                        v-model="key.expires"
                                        :dark="$root.isDark"
                                        :monthChangeOnScroll="false"
                                        :minDate="minDate"
                                        format="yyyy-MM-dd HH:mm"
                                        modelType="yyyy-MM-dd HH:mm:ss"
                                        :required="!noExpire"
                                        :disabled="noExpire"
                                    />
                                </div>
                                <div class="col-6 ms-3">
                                    <div class="form-check mb-0">
                                        <input
                                            id="no-expire"
                                            v-model="noExpire"
                                            class="form-check-input"
                                            type="checkbox"
                                        />
                                        <label
                                            class="form-check-label"
                                            for="no-expire"
                                            >{{ $t("Don't expire") }}</label
                                        >
                                    </div>
                                </div>
                            </div>
                        </div>
                    </div>
                    <div class="custom-modal-footer">
                        <button
                            id="monitor-submit-btn"
                            class="btn btn-primary"
                            type="submit"
                            :disabled="processing"
                        >
                            {{ $t("Generate") }}
                        </button>
                    </div>
                </div>
            </form>
        </div>
    </teleport>

    <teleport to="body">
        <!-- 使用v-if而不是v-show，确保组件完全卸载和重新挂载 -->
        <div
            v-if="visibleKeyModal"
            class="custom-modal-container"
            @click.self="hideKeyModal"
        >
            <div
                class="custom-modal"
                :class="{
                    'custom-modal-show': animationStateKeyModal === 'show',
                }"
            >
                <div class="custom-modal-header">
                    <h5 class="custom-modal-title">
                        {{ $t("Key Added") }}
                    </h5>
                    <button
                        type="button"
                        class="custom-close-button"
                        aria-label="Close"
                        @click="hideKeyModal"
                    >
                        ×
                    </button>
                </div>

                <div class="custom-modal-body">
                    <div class="mb-3">
                        {{ $t("apiKeyAddedMsg") }}
                    </div>
                    <div class="mb-3">
                        <CopyableInput v-model="clearKey" disabled="disabled" />
                    </div>
                </div>

                <div class="custom-modal-footer">
                    <button
                        type="button"
                        class="btn btn-primary"
                        @click="hideKeyModal"
                    >
                        {{ $t("Continue") }}
                    </button>
                </div>
            </div>
        </div>
    </teleport>
</template>

<script lang="ts">
import dayjs from "dayjs";
import Datepicker from "@vuepic/vue-datepicker";
import CopyableInput from "./CopyableInput.vue";

export default {
    components: {
        CopyableInput,
        Datepicker,
    },
    props: {},
    // emits: [ "added" ],
    data() {
        return {
            // 自定义模态框状态 - 添加API密钥对话框
            visible: false,
            animationState: "hidden",
            animationTimeout: null,
            lockShow: false, // 防止重复显示

            // 自定义模态框状态 - 密钥添加成功对话框
            visibleKeyModal: false,
            animationStateKeyModal: "hidden",
            animationTimeoutKeyModal: null,
            lockShowKeyModal: false, // 防止重复显示

            processing: false,
            key: {},
            dark: this.$root.theme === "dark",
            minDate: this.$root.date(dayjs()) + " 00:00",
            clearKey: null,
            noExpire: false,
        };
    },

    mounted() {
        // 不再需要初始化Bootstrap Modal
    },

    beforeUnmount() {
        // 清除可能存在的超时
        this.clearTimeouts();
        // 确保移除事件监听器
        document.removeEventListener("keydown", this.handleEscKey);
        document.removeEventListener("keydown", this.handleEscKeyModal);
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
            if (this.animationTimeoutKeyModal) {
                clearTimeout(this.animationTimeoutKeyModal);
                this.animationTimeoutKeyModal = null;
            }
        },

        /**
         * 处理ESC键按下 - 添加API密钥对话框
         * @param {KeyboardEvent} event 键盘事件
         * @returns {void}
         */
        handleEscKey(event) {
            if (event.key === "Escape") {
                this.hide();
            }
        },

        /**
         * 处理ESC键按下 - 密钥添加成功对话框
         * @param {KeyboardEvent} event 键盘事件
         * @returns {void}
         */
        handleEscKeyModal(event) {
            if (event.key === "Escape") {
                this.hideKeyModal();
            }
        },

        /**
         * 显示添加API密钥对话框
         * @returns {void}
         */
        show() {
            // 如果已经显示或者锁定中，不做任何操作
            if (this.visible || this.lockShow) {
                return;
            }

            // 锁定显示操作，防止重复调用
            this.lockShow = true;

            // 初始化数据
            this.id = null;
            this.key = {
                name: "",
                expires: this.minDate,
                active: 1,
            };

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
         * 隐藏添加API密钥对话框
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
                // 如果没有其他模态框显示，恢复背景滚动
                if (!this.visibleKeyModal) {
                    document.body.style.overflow = "";
                }
            }, 300); // 300ms是过渡动画的时间
        },

        /**
         * 显示密钥添加成功对话框
         * @returns {void}
         */
        showKeyModal() {
            // 如果已经显示或者锁定中，不做任何操作
            if (this.visibleKeyModal || this.lockShowKeyModal) {
                return;
            }

            // 锁定显示操作，防止重复调用
            this.lockShowKeyModal = true;

            // 先显示背景
            this.visibleKeyModal = true;

            // 添加ESC键监听
            document.addEventListener("keydown", this.handleEscKeyModal);

            // 使用nextTick确保DOM已更新
            this.$nextTick(() => {
                // 添加一个小延迟，确保CSS过渡效果正常工作
                this.animationTimeoutKeyModal = setTimeout(() => {
                    this.animationStateKeyModal = "show";
                    // 防止背景滚动
                    document.body.style.overflow = "hidden";
                    // 解除锁定
                    this.lockShowKeyModal = false;
                }, 50);
            });
        },

        /**
         * 隐藏密钥添加成功对话框
         * @returns {void}
         */
        hideKeyModal() {
            // 如果已经隐藏，不做任何操作
            if (!this.visibleKeyModal) {
                return;
            }

            // 先触发隐藏动画
            this.animationStateKeyModal = "hidden";

            // 移除ESC键监听
            document.removeEventListener("keydown", this.handleEscKeyModal);

            // 等待动画完成后再移除元素
            this.animationTimeoutKeyModal = setTimeout(() => {
                this.visibleKeyModal = false;
                // 恢复背景滚动
                document.body.style.overflow = "";
            }, 300); // 300ms是过渡动画的时间
        },

        /**
         * Submit data to server
         * @returns {Promise<void>}
         */
        async submit() {
            this.processing = true;

            if (this.noExpire) {
                this.key.expires = null;
            }

            this.$root.addAPIKey(this.key, async (res) => {
                this.hide();
                this.processing = false;
                if (res.ok) {
                    this.clearKey = res.key;
                    this.showKeyModal();
                    this.clearForm();
                } else {
                    this.$root.toastError(res.msg);
                }
            });
        },

        /**
         * Clear Form inputs
         * @returns {void}
         */
        clearForm() {
            this.key = {
                name: "",
                expires: this.minDate,
                active: 1,
            };
            this.noExpire = false;
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

.shadow-box {
    padding: 20px;
}

textarea {
    min-height: 150px;
}

.dark-calendar::-webkit-calendar-picker-indicator {
    filter: invert(1);
}

.weekday-picker {
    display: flex;
    gap: 10px;

    & > div {
        display: flex;
        flex-direction: column;
        align-items: center;
        width: 40px;

        .form-check-inline {
            margin-right: 0;
        }
    }
}

.day-picker {
    display: flex;
    gap: 10px;
    flex-wrap: wrap;

    & > div {
        display: flex;
        flex-direction: column;
        align-items: center;
        width: 40px;

        .form-check-inline {
            margin-right: 0;
        }
    }
}
</style>
