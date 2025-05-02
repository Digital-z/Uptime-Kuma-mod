<template>
    <teleport to="body">
        <!-- Vue 3.4.2 支持 teleport 功能 -->
        <!-- 使用v-if而不是v-show，确保组件完全卸载和重新挂载 -->
        <div v-if="visible" class="custom-modal-container" @click.self="hide">
            <div
                class="custom-modal custom-modal-xl"
                :class="{ 'custom-modal-show': animationState === 'show' }"
            >
                <div class="custom-modal-header">
                    <h5 class="custom-modal-title">
                        {{ $t("Browser Screenshot") }}
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
                    <img
                        :src="imageURL"
                        alt="screenshot of the website"
                        class="screenshot-img"
                    />
                </div>
            </div>
        </div>
    </teleport>
</template>

<script lang="ts">
export default {
    props: {
        imageURL: {
            type: String,
            required: true,
        },
    },
    data() {
        return {
            visible: false,
            animationState: "hidden",
            animationTimeout: null,
            lockShow: false, // 防止重复显示
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
         * 处理ESC键按下
         * @param {KeyboardEvent} event 键盘事件
         * @returns {void}
         */
        handleEscKey(event) {
            if (event.key === "Escape") {
                this.hide();
            }
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

/* 大尺寸模态框 */
.custom-modal-xl {
    width: 90%;
    max-width: 1140px;
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

/* 截图样式 */
.screenshot-img {
    max-width: 100%;
    height: auto;
    display: block;
    margin: 0 auto;
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

    .custom-close-button {
        color: #f8f9fa;
    }

    .form-text,
    p {
        color: $dark-font-color;
    }
}
</style>
