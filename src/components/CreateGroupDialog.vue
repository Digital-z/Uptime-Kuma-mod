<template>
    <div ref="modal" class="modal fade" tabindex="-1">
        <div class="modal-dialog">
            <div class="modal-content">
                <div class="modal-header">
                    <h5 class="modal-title">
                        {{ $t("New Group") }}
                    </h5>
                    <button
                        type="button"
                        class="btn-close"
                        data-bs-dismiss="modal"
                        aria-label="Close"
                    />
                </div>
                <div class="modal-body">
                    <form @submit.prevent="confirm">
                        <div>
                            <label for="draftGroupName" class="form-label">{{
                                $t("Group Name")
                            }}</label>
                            <input
                                id="draftGroupName"
                                v-model="groupName"
                                type="text"
                                class="form-control"
                                required
                            />
                        </div>
                    </form>
                </div>
                <div class="modal-footer">
                    <button
                        type="button"
                        class="btn btn-secondary"
                        data-bs-dismiss="modal"
                    >
                        {{ $t("Cancel") }}
                    </button>
                    <button
                        type="button"
                        class="btn btn-primary"
                        data-bs-dismiss="modal"
                        :disabled="groupName == '' || groupName == null"
                        @click="confirm"
                    >
                        {{ $t("Confirm") }}
                    </button>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
import { Modal } from "bootstrap";

export default {
    props: {},
    emits: ["added"],
    data: () => ({
        modal: null,
        groupName: null,
    }),
    mounted() {
        this.modal = new Modal(this.$refs.modal);
    },
    beforeUnmount() {
        this.cleanupModal();
    },
    methods: {
        /**
         * Show the confirm dialog
         * @returns {void}
         */
        show() {
            this.modal.show();
        },
        /**
         * Dialog confirmed
         * @returns {void}
         */
        confirm() {
            this.$emit("added", this.groupName);
            this.modal.hide();
        },
        /**
         * Clean up modal and restore scroll behavior
         * @returns {void}
         */
        cleanupModal() {
            if (this.modal) {
                try {
                    this.modal.hide();
                } catch (e) {
                    console.warn("Modal hide failed:", e);
                }
            }
        },
    },
};
</script>

<style lang="scss" scoped>
@import "../assets/vars.scss";

.modal-content {
    background: rgba(255, 255, 255, 0.9) !important;
    backdrop-filter: blur(10px) !important;
    -webkit-backdrop-filter: blur(10px) !important;
    border: 1px solid rgba(255, 255, 255, 0.3) !important;
    border-radius: 12px !important;

    .dark & {
        background: rgba(13, 17, 23, 0.8) !important;
        border: 1px solid rgba(255, 255, 255, 0.1) !important;
    }
}

/* 确保模态框中的输入框有正确的毛玻璃效果 */
.form-control {
    background: rgba(255, 255, 255, 0.2) !important;
    backdrop-filter: blur(8px) !important;
    -webkit-backdrop-filter: blur(8px) !important;
    border: 2px solid rgba(255, 255, 255, 0.5) !important;
    color: inherit !important;
    border-radius: 8px !important;
    padding: 10px 15px !important;
    margin-bottom: 10px !important;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08) !important;
    transition: all 0.3s ease !important;

    &:focus {
        background: rgba(255, 255, 255, 0.3) !important;
        box-shadow: 0 0 0 0.25rem rgba(92, 221, 139, 0.25) !important;
        border-color: rgba(92, 221, 139, 0.5) !important;
    }

    &:hover {
        background: rgba(255, 255, 255, 0.3) !important;
        box-shadow: 0 6px 15px rgba(0, 0, 0, 0.1) !important;
    }

    .dark & {
        background: rgba(13, 17, 23, 0.4) !important;
        border: 2px solid rgba(255, 255, 255, 0.2) !important;
        color: $dark-font-color !important;
        box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2) !important;

        &:focus {
            background: rgba(13, 17, 23, 0.5) !important;
            box-shadow: 0 0 0 0.25rem rgba(92, 221, 139, 0.25) !important;
            border-color: rgba(92, 221, 139, 0.5) !important;
        }

        &:hover {
            background: rgba(13, 17, 23, 0.5) !important;
            box-shadow: 0 6px 15px rgba(0, 0, 0, 0.25) !important;
        }
    }
}

.dark {
    .modal-dialog .form-text,
    .modal-dialog p,
    .modal-title,
    label {
        color: $dark-font-color;
    }
}
</style>
