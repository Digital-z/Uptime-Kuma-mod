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
                            {{ $t("Setup Notification") }}
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
                            <label for="notification-type" class="form-label">{{
                                $t("Notification Type")
                            }}</label>
                            <select
                                id="notification-type"
                                v-model="notification.type"
                                class="form-select"
                            >
                                <option
                                    v-for="(
                                        name, type
                                    ) in notificationNameList.regularList"
                                    :key="type"
                                    :value="type"
                                >
                                    {{ name }}
                                </option>
                                <optgroup :label="$t('notificationRegional')">
                                    <option
                                        v-for="(
                                            name, type
                                        ) in notificationNameList.regionalList"
                                        :key="type"
                                        :value="type"
                                    >
                                        {{ name }}
                                    </option>
                                </optgroup>
                            </select>
                        </div>

                        <div class="mb-3">
                            <label for="notification-name" class="form-label">{{
                                $t("Friendly Name")
                            }}</label>
                            <input
                                id="notification-name"
                                v-model="notification.name"
                                type="text"
                                class="form-control"
                                required
                            />
                        </div>

                        <!-- form body -->
                        <component :is="currentForm" />

                        <div class="mb-3 mt-4">
                            <hr class="dropdown-divider mb-4" />

                            <div class="form-check form-switch">
                                <input
                                    v-model="notification.isDefault"
                                    class="form-check-input"
                                    type="checkbox"
                                />
                                <label class="form-check-label">{{
                                    $t("Default enabled")
                                }}</label>
                            </div>
                            <div class="form-text">
                                {{ $t("enableDefaultNotificationDescription") }}
                            </div>

                            <br />

                            <div class="form-check form-switch">
                                <input
                                    v-model="notification.applyExisting"
                                    class="form-check-input"
                                    type="checkbox"
                                />
                                <label class="form-check-label">{{
                                    $t("Apply on all existing monitors")
                                }}</label>
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
                </div>
            </form>
        </div>
    </teleport>

    <Confirm
        ref="confirmDelete"
        btn-style="btn-danger"
        :yes-text="$t('Yes')"
        :no-text="$t('No')"
        @yes="deleteNotification"
    >
        {{ $t("deleteNotificationMsg") }}
    </Confirm>
</template>

<script>
import Confirm from "./Confirm.vue";
import NotificationFormList from "./notifications";

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
            notificationTypes: Object.keys(NotificationFormList).sort(
                (a, b) => {
                    return a.toLowerCase().localeCompare(b.toLowerCase());
                }
            ),
            notification: {
                name: "",
                /** @type { null | keyof NotificationFormList } */
                type: null,
                isDefault: false,
                // Do not set default value here, please scroll to show()
            },
        };
    },

    computed: {
        currentForm() {
            if (!this.notification.type) {
                return null;
            }
            return NotificationFormList[this.notification.type];
        },

        notificationNameList() {
            let regularList = {
                alerta: "Alerta",
                AlertNow: "AlertNow",
                apprise: this.$t("apprise"),
                Bark: "Bark",
                Bitrix24: "Bitrix24",
                clicksendsms: "ClickSend SMS",
                CallMeBot:
                    "CallMeBot (WhatsApp, Telegram Call, Facebook Messanger)",
                discord: "Discord",
                Elks: "46elks",
                GoogleChat: "Google Chat (Google Workspace)",
                gorush: "Gorush",
                gotify: "Gotify",
                GrafanaOncall: "Grafana Oncall",
                HeiiOnCall: "Heii On-Call",
                HomeAssistant: "Home Assistant",
                Keep: "Keep",
                Kook: "Kook",
                line: "LINE Messenger",
                LineNotify: "LINE Notify",
                lunasea: "LunaSea",
                matrix: "Matrix",
                mattermost: "Mattermost",
                nostr: "Nostr",
                ntfy: "Ntfy",
                octopush: "Octopush",
                OneChat: "OneChat",
                OneBot: "OneBot",
                Onesender: "Onesender",
                Opsgenie: "Opsgenie",
                PagerDuty: "PagerDuty",
                PagerTree: "PagerTree",
                pumble: "Pumble",
                pushbullet: "Pushbullet",
                PushByTechulus: "Push by Techulus",
                pushover: "Pushover",
                pushy: "Pushy",
                "rocket.chat": "Rocket.Chat",
                signal: "Signal",
                SIGNL4: "SIGNL4",
                slack: "Slack",
                squadcast: "SquadCast",
                SMSEagle: "SMSEagle",
                SMSPartner: "SMS Partner",
                smtp: this.$t("smtp"),
                stackfield: "Stackfield",
                teams: "Microsoft Teams",
                telegram: "Telegram",
                threema: "Threema",
                twilio: "Twilio",
                Splunk: "Splunk",
                webhook: "Webhook",
                GoAlert: "GoAlert",
                ZohoCliq: "ZohoCliq",
                SevenIO: "SevenIO",
                whapi: "WhatsApp (Whapi)",
                waha: "WhatsApp (WAHA)",
                gtxmessaging: "GtxMessaging",
                Cellsynt: "Cellsynt",
                SendGrid: "SendGrid",
            };

            // Put notifications here if it's not supported in most regions or its documentation is not in English
            let regionalList = {
                AliyunSMS: "AliyunSMS (阿里云短信服务)",
                DingDing: "DingDing (钉钉自定义机器人)",
                Feishu: "Feishu (飞书)",
                FlashDuty: "FlashDuty (快猫星云)",
                FreeMobile: "FreeMobile (mobile.free.fr)",
                PushDeer: "PushDeer",
                promosms: "PromoSMS",
                serwersms: "SerwerSMS.pl",
                SMSManager: "SmsManager (smsmanager.cz)",
                WeCom: "WeCom (企业微信群机器人)",
                ServerChan: "ServerChan (Server酱)",
                PushPlus: "PushPlus (推送加)",
                smsc: "SMSC",
                WPush: "WPush(wpush.cn)",
                YZJ: "YZJ (云之家自定义机器人)",
                SMSPlanet: "SMSPlanet.pl",
            };

            // Sort by notification name
            // No idea how, but it works
            // https://stackoverflow.com/questions/1069666/sorting-object-property-by-values
            let sort = (list2) => {
                return Object.entries(list2)
                    .sort(([, a], [, b]) => a.localeCompare(b))
                    .reduce(
                        (r, [k, v]) => ({
                            ...r,
                            [k]: v,
                        }),
                        {}
                    );
            };

            return {
                regularList: sort(regularList),
                regionalList: sort(regionalList),
            };
        },

        notificationFullNameList() {
            let list = {};
            for (let [key, value] of Object.entries(
                this.notificationNameList.regularList
            )) {
                list[key] = value;
            }
            for (let [key, value] of Object.entries(
                this.notificationNameList.regionalList
            )) {
                list[key] = value;
            }
            return list;
        },
    },

    watch: {
        "notification.type"(to, from) {
            let oldName;
            if (from) {
                oldName = this.getUniqueDefaultName(from);
            } else {
                oldName = "";
            }

            if (!this.notification.name || this.notification.name === oldName) {
                this.notification.name = this.getUniqueDefaultName(to);
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
         * Show dialog to confirm deletion
         * @returns {void}
         */
        deleteConfirm() {
            this.hide();
            this.$refs.confirmDelete.show();
        },

        /**
         * 显示对话框
         * @param {number} notificationID ID of notification to show
         * @returns {void}
         */
        show(notificationID) {
            // 如果已经显示或者锁定中，不做任何操作
            if (this.visible || this.lockShow) {
                return;
            }

            // 锁定显示操作，防止重复调用
            this.lockShow = true;

            // 加载通知数据
            if (notificationID) {
                this.id = notificationID;

                for (let n of this.$root.notificationList) {
                    if (n.id === notificationID) {
                        this.notification = JSON.parse(n.config);
                        break;
                    }
                }
            } else {
                this.id = null;
                this.notification = {
                    name: "",
                    type: "telegram",
                    isDefault: false,
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
         * Submit the form to the server
         * @returns {void}
         */
        submit() {
            this.processing = true;
            this.$root
                .getSocket()
                .emit("addNotification", this.notification, this.id, (res) => {
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
         * Test the notification endpoint
         * @returns {void}
         */
        test() {
            this.processing = true;
            this.$root
                .getSocket()
                .emit("testNotification", this.notification, (res) => {
                    this.$root.toastRes(res);
                    this.processing = false;
                });
        },

        /**
         * Delete the notification endpoint
         * @returns {void}
         */
        deleteNotification() {
            this.processing = true;
            this.$root
                .getSocket()
                .emit("deleteNotification", this.id, (res) => {
                    this.$root.toastRes(res);
                    this.processing = false;

                    if (res.ok) {
                        this.hide();
                    }
                });
        },
        /**
         * Get a unique default name for the notification
         * @param {keyof NotificationFormList} notificationKey
         * Notification to retrieve
         * @returns {string} Default name
         */
        getUniqueDefaultName(notificationKey) {
            let index = 1;
            let name = "";
            do {
                name = this.$t("defaultNotificationName", {
                    notification: this.notificationFullNameList[notificationKey]
                        .replace(/\(.+\)/, "")
                        .trim(),
                    number: index++,
                });
            } while (
                this.$root.notificationList.find((it) => it.name === name)
            );
            return name;
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
