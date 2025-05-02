<template>
    <teleport to="body">
        <!-- Vue 3.4.2 支持 teleport 功能 -->
        <!-- 使用v-if而不是v-show，确保组件完全卸载和重新挂载 -->
        <div
            v-if="visible"
            class="custom-modal-container"
            @click.self="handleBackdropClick"
        >
            <div
                class="custom-modal"
                :class="{ 'custom-modal-show': animationState === 'show' }"
            >
                <form @submit.prevent="submit">
                    <div class="custom-modal-header">
                        <h5 class="custom-modal-title">
                            {{ $t("Add a Remote Browser") }}
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
                            <label
                                for="remote-browser-name"
                                class="form-label"
                                >{{ $t("Friendly Name") }}</label
                            >
                            <input
                                id="remote-browser-name"
                                v-model="remoteBrowser.name"
                                type="text"
                                class="form-control"
                                required
                            />
                        </div>

                        <div class="mb-3">
                            <label
                                for="remote-browser-url"
                                class="form-label"
                                >{{ $t("URL") }}</label
                            >
                            <input
                                id="remote-browser-url"
                                v-model="remoteBrowser.url"
                                type="text"
                                class="form-control"
                                required
                            />

                            <div class="form-text mt-3">
                                {{ $t("Examples") }}:
                                <ul>
                                    <li>
                                        ws://chrome.browserless.io/playwright?token=YOUR-API-TOKEN
                                    </li>
                                </ul>
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
                            type="button"
                            class="btn btn-warning"
                            :disabled="processing"
                            @click="test"
                        >
                            {{ $t("Test") }}
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
                </form>
            </div>
        </div>
    </teleport>

    <Confirm
        ref="confirmDelete"
        btn-style="btn-danger"
        :yes-text="$t('Yes')"
        :no-text="$t('No')"
        @yes="deleteDockerHost"
    >
        {{ $t("deleteRemoteBrowserMessage") }}
    </Confirm>
</template>

<script>
import Confirm from "./Confirm.vue";

export default {
    components: {
        Confirm,
    },
    props: {},
    emits: ["added"],
    data() {
        return {
            visible: false,
            animationState: "hidden",
            animationTimeout: null,
            lockShow: false, // 防止重复显示
            processing: false,
            id: null,
            remoteBrowser: {
                name: "",
                url: "",
                // 不要在这里设置默认值，请滚动到 show() 方法
            },
        };
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
         * 显示对话框
         * @param {number} remoteBrowserID 要显示的远程浏览器ID
         * @returns {void}
         */
        show(remoteBrowserID) {
            // 如果已经显示或者锁定中，不做任何操作
            if (this.visible || this.lockShow) {
                return;
            }

            // 锁定显示操作，防止重复调用
            this.lockShow = true;

            // 处理远程浏览器数据
            if (remoteBrowserID) {
                let found = false;

                this.id = remoteBrowserID;

                for (let n of this.$root.remoteBrowserList) {
                    if (n.id === remoteBrowserID) {
                        this.remoteBrowser = n;
                        found = true;
                        break;
                    }
                }

                if (!found) {
                    this.$root.toastError(this.$t("Remote Browser not found!"));
                    this.lockShow = false;
                    return;
                }
            } else {
                this.id = null;
                this.remoteBrowser = {
                    name: "",
                    url: "",
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
         * 处理背景点击
         * @returns {void}
         */
        handleBackdropClick() {
            this.hide();
        },

        /**
         * 确认删除远程浏览器
         * @returns {void}
         */
        deleteConfirm() {
            this.hide();
            this.$refs.confirmDelete.show();
        },

        /**
         * 添加远程浏览器
         * @returns {void}
         */
        submit() {
            this.processing = true;
            this.$root
                .getSocket()
                .emit(
                    "addRemoteBrowser",
                    this.remoteBrowser,
                    this.id,
                    (res) => {
                        this.$root.toastRes(res);
                        this.processing = false;

                        if (res.ok) {
                            this.hide();

                            // 发出添加事件，不发出编辑事件
                            if (!this.id) {
                                this.$emit("added", res.id);
                            }
                        }
                    }
                );
        },

        /**
         * 测试远程浏览器
         * @returns {void}
         */
        test() {
            this.processing = true;
            this.$root
                .getSocket()
                .emit("testRemoteBrowser", this.remoteBrowser, (res) => {
                    this.$root.toastRes(res);
                    this.processing = false;
                });
        },

        /**
         * 删除远程浏览器
         * @returns {void}
         */
        deleteDockerHost() {
            this.processing = true;
            this.$root
                .getSocket()
                .emit("deleteRemoteBrowser", this.id, (res) => {
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
}

/* 模态框底部 */
.custom-modal-footer {
    display: flex;
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

    .custom-modal-header,
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
