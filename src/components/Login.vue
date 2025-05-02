<template>
    <div class="form-container">
        <div class="form glassmorphism">
            <form @submit.prevent="submit">
                <h1 class="h3 mb-3 fw-normal" />

                <div v-if="!tokenRequired" class="form-floating">
                    <input
                        id="floatingInput"
                        v-model="username"
                        type="text"
                        class="form-control glass-input"
                        placeholder=" "
                        autocomplete="username"
                        required
                    />
                    <label for="floatingInput">{{ $t("Username") }}</label>
                </div>

                <div v-if="!tokenRequired" class="form-floating mt-3">
                    <input
                        id="floatingPassword"
                        v-model="password"
                        type="password"
                        class="form-control glass-input"
                        placeholder=" "
                        autocomplete="current-password"
                        required
                    />
                    <label for="floatingPassword">{{ $t("Password") }}</label>
                </div>

                <div v-if="tokenRequired">
                    <div class="form-floating mt-3">
                        <input
                            id="otp"
                            v-model="token"
                            type="text"
                            maxlength="6"
                            class="form-control glass-input"
                            placeholder=" "
                            autocomplete="one-time-code"
                            required
                        />
                        <label for="otp">{{ $t("Token") }}</label>
                    </div>
                </div>

                <div
                    class="form-check mb-3 mt-3 d-flex justify-content-center pe-4"
                >
                    <div class="form-check">
                        <input
                            id="remember"
                            v-model="$root.remember"
                            type="checkbox"
                            value="remember-me"
                            class="form-check-input"
                        />

                        <label class="form-check-label" for="remember">
                            {{ $t("Remember me") }}
                        </label>
                    </div>
                </div>
                <button
                    class="w-100 btn btn-primary glass-button"
                    type="submit"
                    :disabled="processing"
                >
                    {{ $t("Login") }}
                </button>

                <div
                    v-if="res && !res.ok"
                    class="alert alert-danger mt-3"
                    role="alert"
                >
                    {{ $t(res.msg) }}
                </div>
            </form>
        </div>
    </div>
</template>

<script>
export default {
    data() {
        return {
            processing: false,
            username: "",
            password: "",
            token: "",
            res: null,
            tokenRequired: false,
        };
    },

    mounted() {
        document.title += " - Login";
        document.body.classList.add("login-background");
    },

    unmounted() {
        document.title = document.title.replace(" - Login", "");
        document.body.classList.remove("login-background");
    },

    methods: {
        /**
         * Submit the user details and attempt to log in
         * @returns {void}
         */
        submit() {
            this.processing = true;

            this.$root.login(
                this.username,
                this.password,
                this.token,
                (res) => {
                    this.processing = false;

                    if (res.tokenRequired) {
                        this.tokenRequired = true;
                    } else {
                        this.res = res;
                    }
                }
            );
        },
    },
};
</script>

<style lang="scss" scoped>
.form-container {
    display: flex;
    align-items: center;
    padding-top: 40px;
    padding-bottom: 40px;
}

/* 毛玻璃效果容器样式 */
.glassmorphism {
    background: rgba(255, 255, 255, 0.2); /* 半透明白色背景 */
    backdrop-filter: blur(10px); /* 背景模糊效果 */
    -webkit-backdrop-filter: blur(10px); /* Safari浏览器兼容 */
    border-radius: 16px; /* 圆角边框 */
    border: 1px solid rgba(255, 255, 255, 0.3); /* 半透明边框 */
    box-shadow: 0 4px 30px rgba(0, 0, 0, 0.1); /* 轻微阴影效果 */
}

/* 玻璃质感输入框样式 */
.glass-input {
    background: transparent !important; /* 完全透明背景 */
    backdrop-filter: blur(8px) !important; /* 背景模糊效果 */
    -webkit-backdrop-filter: blur(8px) !important; /* Safari浏览器兼容 */
    border: 0.9px solid rgba(255, 255, 255, 0.5) !important; /* 半透明边框 */
    color: inherit !important; /* 继承父元素文字颜色 */
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08) !important;
    transition: all 0.3s ease !important;

    &:focus {
        background: transparent !important;
        box-shadow: 0 0 0 0.25rem rgba(92, 221, 139, 0.25) !important;
        border-color: rgba(92, 221, 139, 0.5) !important;
    }

    &:hover {
        background: transparent !important;
        box-shadow: 0 6px 15px rgba(0, 0, 0, 0.1) !important;
    }

    /* 确保在输入内容后仍然保持透明背景 */
    &:not(:placeholder-shown) {
        background: transparent !important;
    }

    .dark & {
        background: transparent !important;
        border: 2px solid rgba(255, 255, 255, 0.2) !important;
        color: #b1b8c0 !important;
        box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2) !important;

        &:focus {
            background: transparent !important;
            box-shadow: 0 0 0 0.25rem rgba(92, 221, 139, 0.25) !important;
            border-color: rgba(92, 221, 139, 0.5) !important;
        }

        &:hover {
            background: transparent !important;
            box-shadow: 0 6px 15px rgba(0, 0, 0, 0.25) !important;
        }

        /* 确保在暗模式下输入内容后仍然保持透明背景 */
        &:not(:placeholder-shown) {
            background: transparent !important;
        }
    }
}

/* 玻璃质感按钮样式 */
.glass-button {
    background: rgba(95, 207, 128, 0.8) !important; /* 半透明绿色背景 */
    backdrop-filter: blur(5px); /* 背景模糊效果 */
    -webkit-backdrop-filter: blur(5px); /* Safari浏览器兼容 */
    border: none !important; /* 移除边框 */
    transition: background 0.3s ease !important; /* 平滑过渡效果 */

    &:hover:not(:disabled) {
        background: rgba(95, 207, 128, 0.9) !important; /* 悬停时加深背景色 */
    }
}

.form-floating {
    > label {
        padding-left: 1.3rem;
        z-index: 1; /* 确保标签显示在输入框上方 */
        color: rgba(0, 0, 0, 0.7); /* 标签文字颜色 */
    }

    > .form-control {
        padding-left: 1.3rem;
        height: auto;

        /* 确保浮动标签状态下背景仍然透明 */
        &:focus,
        &:not(:placeholder-shown) {
            background: transparent !important;

            ~ label {
                opacity: 1;
                transform: scale(0.85) translateY(-0.5rem) translateX(0.15rem); /* 标签浮动动画 */
                background: transparent; /* 透明背景 */
            }
        }
    }
}

.form {
    width: 100%;
    max-width: 330px;
    padding: 15px;
    margin: auto;
    text-align: center;
}

/* 登录页面全局背景样式 */
:global(.login-background) {
    background: linear-gradient(
        135deg,
        #67b26f 0%,
        #4ca2cd 100%
    ); /* 渐变背景 */
    background-size: cover; /* 覆盖整个视口 */
    background-attachment: fixed; /* 固定背景 */
    min-height: 100vh; /* 最小高度为视口高度 */
}

/* 记住我复选框样式 */
.form-check-input {
    width: 18px;
    height: 18px;
    background-color: transparent !important;
    border: 2px solid rgba(255, 255, 255, 0.5);
    border-radius: 4px;
    cursor: pointer;
    transition: all 0.2s ease;
    margin-right: 6px;
    position: relative;

    &:checked {
        background-color: rgba(
            92,
            221,
            139,
            0.8
        ) !important; /* 较深的绿色背景 */
        border: 2px solid rgba(92, 221, 139, 0.9) !important; /* 加深边框颜色 */
        box-shadow: 0 0 8px rgba(92, 221, 139, 0.4) !important; /* 添加轻微发光效果 */
    }

    &:focus {
        box-shadow: 0 0 0 0.25rem rgba(92, 221, 139, 0.25);
        border-color: rgba(92, 221, 139, 0.5);
        background-color: transparent !important;
    }
}
</style>
