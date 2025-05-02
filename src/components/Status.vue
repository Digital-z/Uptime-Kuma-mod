<template>
    <span class="badge rounded-pill glass-badge" :class="'bg-' + color">{{
        text
    }}</span>
</template>

<script>
export default {
    props: {
        /** Current status of monitor */
        status: {
            type: Number,
            default: 0,
        },
    },

    computed: {
        color() {
            if (this.status === 0) {
                return "danger";
            }

            if (this.status === 1) {
                return "primary";
            }

            if (this.status === 2) {
                return "warning";
            }

            if (this.status === 3) {
                return "maintenance";
            }

            return "secondary";
        },

        text() {
            if (this.status === 0) {
                return this.$t("Down");
            }

            if (this.status === 1) {
                return this.$t("Up");
            }

            if (this.status === 2) {
                return this.$t("Pending");
            }

            if (this.status === 3) {
                return this.$t("statusMaintenance");
            }

            return this.$t("Unknown");
        },
    },
};
</script>

<style lang="scss" scoped>
@import "../assets/vars.scss";

/* 基础样式 */
span {
    min-width: 64px;
    letter-spacing: 0.5px;
    font-weight: 500;
}

/* 毛玻璃状态标签 */
.glass-badge {
    backdrop-filter: blur($glass-blur); /* 背景模糊效果 */
    -webkit-backdrop-filter: blur($glass-blur); /* Safari浏览器兼容 */
    border: 2px solid rgba(0, 0, 0, 0.2) !important; /* 更明显的边框 */
    box-shadow: $glass-shadow; /* 轻微阴影 */
    transition: all 0.3s ease; /* 平滑过渡效果 */

    .dark & {
        border: 2px solid rgba(255, 255, 255, 0.2) !important; /* 暗色模式下更明显的边框 */
    }
}

/* 状态标签背景颜色透明化 */
.bg-primary {
    background-color: rgba(
        92,
        221,
        139,
        0.8
    ) !important; /* 正常状态-绿色半透明 */

    &:hover {
        background-color: rgba(
            92,
            221,
            139,
            0.9
        ) !important; /* 悬停时加深背景色 */
        transform: translateY(-1px); /* 轻微上浮效果 */
    }
}

.bg-danger {
    background-color: rgba(
        220,
        53,
        69,
        0.8
    ) !important; /* 故障状态-红色半透明 */

    &:hover {
        background-color: rgba(
            220,
            53,
            69,
            0.9
        ) !important; /* 悬停时加深背景色 */
        transform: translateY(-1px); /* 轻微上浮效果 */
    }
}

.bg-warning {
    background-color: rgba(
        248,
        163,
        6,
        0.8
    ) !important; /* 警告状态-黄色半透明 */

    &:hover {
        background-color: rgba(
            248,
            163,
            6,
            0.9
        ) !important; /* 悬停时加深背景色 */
        transform: translateY(-1px); /* 轻微上浮效果 */
    }
}

.bg-maintenance {
    background-color: rgba(
        23,
        71,
        245,
        0.8
    ) !important; /* 维护状态-蓝色半透明 */

    &:hover {
        background-color: rgba(
            23,
            71,
            245,
            0.9
        ) !important; /* 悬停时加深背景色 */
        transform: translateY(-1px); /* 轻微上浮效果 */
    }
}

.bg-secondary {
    background-color: rgba(
        108,
        117,
        125,
        0.8
    ) !important; /* 未知状态-灰色半透明 */

    &:hover {
        background-color: rgba(
            108,
            117,
            125,
            0.9
        ) !important; /* 悬停时加深背景色 */
        transform: translateY(-1px); /* 轻微上浮效果 */
    }
}
</style>
