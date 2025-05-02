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
                            {{ $t("Setup 2FA") }}
                            <span
                                v-if="twoFAStatus == true"
                                class="badge bg-primary"
                                >{{ $t("Active") }}</span
                            >
                            <span
                                v-if="twoFAStatus == false"
                                class="badge bg-primary"
                                >{{ $t("Inactive") }}</span
                            >
                        </h5>
                        <button
                            :disabled="processing"
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
                            <div
                                v-if="uri && twoFAStatus == false"
                                class="mx-auto text-center"
                                style="width: 210px"
                            >
                                <vue-qrcode
                                    :key="uri"
                                    :value="uri"
                                    type="image/png"
                                    :quality="1"
                                    :color="{ light: '#ffffffff' }"
                                />
                                <button
                                    v-show="!showURI"
                                    type="button"
                                    class="btn btn-outline-primary btn-sm mt-2"
                                    @click="showURI = true"
                                >
                                    {{ $t("Show URI") }}
                                </button>
                            </div>
                            <p
                                v-if="showURI && twoFAStatus == false"
                                class="text-break mt-2"
                            >
                                {{ uri }}
                            </p>

                            <div
                                v-if="!(uri && twoFAStatus == false)"
                                class="mb-3"
                            >
                                <label
                                    for="current-password"
                                    class="form-label"
                                >
                                    {{ $t("Current Password") }}
                                </label>
                                <input
                                    id="current-password"
                                    v-model="currentPassword"
                                    type="password"
                                    class="form-control"
                                    autocomplete="current-password"
                                    required
                                />
                            </div>

                            <button
                                v-if="uri == null && twoFAStatus == false"
                                class="btn btn-primary"
                                type="button"
                                @click="prepare2FA()"
                            >
                                {{ $t("Enable 2FA") }}
                            </button>

                            <button
                                v-if="twoFAStatus == true"
                                class="btn btn-danger"
                                type="button"
                                :disabled="processing"
                                @click="confirmDisableTwoFA()"
                            >
                                {{ $t("Disable 2FA") }}
                            </button>

                            <div
                                v-if="uri && twoFAStatus == false"
                                class="mt-3"
                            >
                                <label for="basic-url" class="form-label">{{
                                    $t("twoFAVerifyLabel")
                                }}</label>
                                <div class="input-group">
                                    <input
                                        v-model="token"
                                        type="text"
                                        maxlength="6"
                                        class="form-control"
                                        autocomplete="one-time-code"
                                        required
                                    />
                                    <button
                                        class="btn btn-outline-primary"
                                        type="button"
                                        @click="verifyToken()"
                                    >
                                        {{ $t("Verify Token") }}
                                    </button>
                                </div>
                                <p
                                    v-show="tokenValid"
                                    class="mt-2"
                                    style="color: green"
                                >
                                    {{ $t("tokenValidSettingsMsg") }}
                                </p>
                            </div>
                        </div>
                    </div>

                    <div
                        v-if="uri && twoFAStatus == false"
                        class="custom-modal-footer"
                    >
                        <button
                            type="submit"
                            class="btn btn-primary"
                            :disabled="processing || tokenValid == false"
                            @click="confirmEnableTwoFA()"
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
        ref="confirmEnableTwoFA"
        btn-style="btn-danger"
        :yes-text="$t('Yes')"
        :no-text="$t('No')"
        @yes="save2FA"
    >
        {{ $t("confirmEnableTwoFAMsg") }}
    </Confirm>

    <Confirm
        ref="confirmDisableTwoFA"
        btn-style="btn-danger"
        :yes-text="$t('Yes')"
        :no-text="$t('No')"
        @yes="disable2FA"
    >
        {{ $t("confirmDisableTwoFAMsg") }}
    </Confirm>
</template>

<script lang="ts">
import Confirm from "./Confirm.vue";
import VueQrcode from "vue-qrcode";

export default {
    components: {
        Confirm,
        VueQrcode,
    },
    props: {},
    data() {
        return {
            // 自定义模态框状态
            visible: false,
            animationState: "hidden",
            animationTimeout: null,
            lockShow: false, // 防止重复显示

            currentPassword: "",
            processing: false,
            uri: null,
            tokenValid: false,
            twoFAStatus: null,
            token: null,
            showURI: false,
        };
    },
    mounted() {
        this.getStatus();
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
         * 显示对话框
         * @returns {void}
         */
        show() {
            // 如果已经显示或者锁定中，不做任何操作
            if (this.visible || this.lockShow) {
                return;
            }

            // 锁定显示操作，防止重复调用
            this.lockShow = true;

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
         * Show dialog to confirm enabling 2FA
         * @returns {void}
         */
        confirmEnableTwoFA() {
            this.$refs.confirmEnableTwoFA.show();
        },

        /**
         * Show dialog to confirm disabling 2FA
         * @returns {void}
         */
        confirmDisableTwoFA() {
            this.$refs.confirmDisableTwoFA.show();
        },

        /**
         * Prepare 2FA configuration
         * @returns {void}
         */
        prepare2FA() {
            this.processing = true;

            this.$root
                .getSocket()
                .emit("prepare2FA", this.currentPassword, (res) => {
                    this.processing = false;

                    if (res.ok) {
                        this.uri = res.uri;
                    } else {
                        this.$root.toastError(res.msg);
                    }
                });
        },

        /**
         * Save the current 2FA configuration
         * @returns {void}
         */
        save2FA() {
            this.processing = true;

            this.$root
                .getSocket()
                .emit("save2FA", this.currentPassword, (res) => {
                    this.processing = false;

                    if (res.ok) {
                        this.$root.toastRes(res);
                        this.getStatus();
                        this.currentPassword = "";
                        this.hide();
                    } else {
                        this.$root.toastError(res.msg);
                    }
                });
        },

        /**
         * Disable 2FA for this user
         * @returns {void}
         */
        disable2FA() {
            this.processing = true;

            this.$root
                .getSocket()
                .emit("disable2FA", this.currentPassword, (res) => {
                    this.processing = false;

                    if (res.ok) {
                        this.$root.toastRes(res);
                        this.getStatus();
                        this.currentPassword = "";
                        this.hide();
                    } else {
                        this.$root.toastError(res.msg);
                    }
                });
        },

        /**
         * Verify the token generated by the user
         * @returns {void}
         */
        verifyToken() {
            this.$root
                .getSocket()
                .emit(
                    "verifyToken",
                    this.token,
                    this.currentPassword,
                    (res) => {
                        if (res.ok) {
                            this.tokenValid = res.valid;
                        } else {
                            this.$root.toastError(res.msg);
                        }
                    }
                );
        },

        /**
         * Get current status of 2FA
         * @returns {void}
         */
        getStatus() {
            this.$root.getSocket().emit("twoFAStatus", (res) => {
                if (res.ok) {
                    this.twoFAStatus = res.status;
                } else {
                    this.$root.toastError(res.msg);
                }
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
