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
                            {{ $t("Setup Proxy") }}
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
                            <label for="proxy-protocol" class="form-label">{{
                                $t("Proxy Protocol")
                            }}</label>
                            <select
                                id="proxy-protocol"
                                v-model="proxy.protocol"
                                class="form-select"
                            >
                                <option value="https">HTTPS</option>
                                <option value="http">HTTP</option>
                                <option value="socks">SOCKS</option>
                                <option value="socks5">SOCKS v5</option>
                                <option value="socks5h">SOCKS v5 (+DNS)</option>
                                <option value="socks4">SOCKS v4</option>
                            </select>
                        </div>

                        <div class="mb-3">
                            <label for="proxy-host" class="form-label">{{
                                $t("Proxy Server")
                            }}</label>
                            <div class="d-flex">
                                <input
                                    id="proxy-host"
                                    v-model="proxy.host"
                                    type="text"
                                    class="form-control"
                                    required
                                    :placeholder="$t('Server Address')"
                                />
                                <input
                                    v-model="proxy.port"
                                    type="number"
                                    class="form-control ms-2"
                                    style="width: 100px"
                                    required
                                    min="1"
                                    max="65535"
                                    :placeholder="$t('Port')"
                                />
                            </div>
                        </div>

                        <div class="mb-3">
                            <div class="form-check form-switch">
                                <input
                                    id="mark-auth"
                                    v-model="proxy.auth"
                                    class="form-check-input"
                                    type="checkbox"
                                />
                                <label
                                    for="mark-auth"
                                    class="form-check-label"
                                    >{{
                                        $t("Proxy server has authentication")
                                    }}</label
                                >
                            </div>
                        </div>

                        <div v-if="proxy.auth" class="mb-3">
                            <label for="proxy-username" class="form-label">{{
                                $t("User")
                            }}</label>
                            <input
                                id="proxy-username"
                                v-model="proxy.username"
                                type="text"
                                class="form-control"
                                required
                            />
                        </div>

                        <div v-if="proxy.auth" class="mb-3">
                            <label for="proxy-password" class="form-label">{{
                                $t("Password")
                            }}</label>
                            <input
                                id="proxy-password"
                                v-model="proxy.password"
                                type="password"
                                class="form-control"
                                required
                            />
                        </div>

                        <div class="mb-3 mt-4">
                            <hr class="dropdown-divider mb-4" />

                            <div class="form-check form-switch">
                                <input
                                    id="mark-active"
                                    v-model="proxy.active"
                                    class="form-check-input"
                                    type="checkbox"
                                />
                                <label
                                    for="mark-active"
                                    class="form-check-label"
                                    >{{ $t("enabled") }}</label
                                >
                            </div>
                            <div class="form-text">
                                {{ $t("enableProxyDescription") }}
                            </div>

                            <br />

                            <div class="form-check form-switch">
                                <input
                                    id="mark-default"
                                    v-model="proxy.default"
                                    class="form-check-input"
                                    type="checkbox"
                                />
                                <label
                                    for="mark-default"
                                    class="form-check-label"
                                    >{{ $t("setAsDefault") }}</label
                                >
                            </div>
                            <div class="form-text">
                                {{ $t("setAsDefaultProxyDescription") }}
                            </div>

                            <br />

                            <div class="form-check form-switch">
                                <input
                                    id="apply-existing"
                                    v-model="proxy.applyExisting"
                                    class="form-check-input"
                                    type="checkbox"
                                />
                                <label
                                    class="form-check-label"
                                    for="apply-existing"
                                    >{{
                                        $t("Apply on all existing monitors")
                                    }}</label
                                >
                            </div>
                        </div>
                    </div>

                    <div class="custom-modal-footer">
                        <button
                            v-if="id"
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
        @yes="deleteProxy"
    >
        {{ $t("deleteProxyMsg") }}
    </Confirm>
</template>

<script lang="ts">
import Confirm from "./Confirm.vue";

export default {
    components: {
        Confirm,
    },
    props: {},
    emits: ["added"],
    data() {
        return {
            // 自定义模态框状态
            visible: false,
            animationState: "hidden",
            animationTimeout: null,
            lockShow: false, // 防止重复显示

            processing: false,
            id: null,
            proxy: {
                protocol: null,
                host: null,
                port: null,
                auth: false,
                username: null,
                password: null,
                active: false,
                default: false,
                applyExisting: false,
            },
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
         * Show dialog to confirm deletion
         * @returns {void}
         */
        deleteConfirm() {
            this.hide();
            this.$refs.confirmDelete.show();
        },

        /**
         * 显示代理设置对话框
         * @param {number} proxyID ID of proxy to show
         * @returns {void}
         */
        show(proxyID) {
            // 如果已经显示或者锁定中，不做任何操作
            if (this.visible || this.lockShow) {
                return;
            }

            // 锁定显示操作，防止重复调用
            this.lockShow = true;

            // 加载代理数据
            if (proxyID) {
                this.id = proxyID;

                for (let proxy of this.$root.proxyList) {
                    if (proxy.id === proxyID) {
                        this.proxy = proxy;
                        break;
                    }
                }
            } else {
                this.id = null;
                this.proxy = {
                    protocol: "https",
                    host: null,
                    port: null,
                    auth: false,
                    username: null,
                    password: null,
                    active: true,
                    default: false,
                    applyExisting: false,
                };
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
         * Submit form data for saving
         * @returns {void}
         */
        submit() {
            this.processing = true;
            this.$root
                .getSocket()
                .emit("addProxy", this.proxy, this.id, (res) => {
                    this.$root.toastRes(res);
                    this.processing = false;

                    if (res.ok) {
                        this.hide();

                        // Emit added event, doesn't emit edit.
                        if (!this.id) {
                            this.$emit("added", res.id);
                        }
                    }
                });
        },

        /**
         * Delete this proxy
         * @returns {void}
         */
        deleteProxy() {
            this.processing = true;
            this.$root.getSocket().emit("deleteProxy", this.id, (res) => {
                this.$root.toastRes(res);
                this.processing = false;

                if (res.ok) {
                    this.hide();
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
