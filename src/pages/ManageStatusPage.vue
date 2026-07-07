<template>
    <transition name="slide-fade" appear>
        <div>
            <div class="d-flex align-items-center justify-content-between mb-3">
                <h1 class="mb-0">
                    {{ $t("Status Pages") }}
                </h1>
                <button class="btn btn-primary" @click="showAddDialog">
                    <font-awesome-icon icon="plus" />
                    {{ $t("New Status Page") }}
                </button>
            </div>

            <div class="shadow-box">
                <template v-if="$root.statusPageListLoaded">
                    <span
                        v-if="Object.keys($root.statusPageList).length === 0"
                        class="d-flex align-items-center justify-content-center my-3"
                    >
                        {{ $t("No status pages") }}
                    </span>

                    <!-- use <a> instead of <router-link>, because the heartbeat won't load. -->
                    <a
                        v-for="statusPage in $root.statusPageList"
                        :key="statusPage.slug"
                        :href="'/status/' + statusPage.slug"
                        class="item"
                    >
                        <img :src="icon(statusPage.icon)" alt class="logo me-2" />
                        <div class="info">
                            <div class="title">{{ statusPage.title }}</div>
                            <div class="slug">/status/{{ statusPage.slug }}</div>
                        </div>
                        <div class="actions">
                            <button
                                class="btn btn-danger delete-status-page"
                                @click.stop.prevent="deleteDialog(statusPage.slug)"
                            >
                                <font-awesome-icon icon="trash" />
                                <span>{{ $t("Delete") }}</span>
                            </button>
                        </div>
                    </a>
                </template>
                <div v-else class="d-flex align-items-center justify-content-center my-3 spinner">
                    <font-awesome-icon icon="spinner" size="2x" spin />
                </div>
            </div>
        </div>
    </transition>
    <Confirm
        ref="confirmDelete"
        btn-style="btn-danger"
        :yes-text="$t('Yes')"
        :no-text="$t('No')"
        @yes="deleteStatusPage"
    >
        {{ $t("deleteStatusPageMsg") }}
    </Confirm>

    <!-- Create Status Page dialog -->
    <div ref="addModal" class="modal fade" tabindex="-1">
        <div class="modal-dialog">
            <form class="modal-content" @submit.prevent="createStatusPage">
                <div class="modal-header">
                    <h5 class="modal-title">{{ $t("Add New Status Page") }}</h5>
                    <button type="button" class="btn-close" @click="hideAddDialog" />
                </div>
                <div class="modal-body">
                    <div class="mb-3">
                        <label for="new-sp-name" class="form-label">{{ $t("Name") }}</label>
                        <input
                            id="new-sp-name"
                            ref="nameInput"
                            v-model="newTitle"
                            type="text"
                            class="form-control"
                            required
                            data-testid="name-input"
                            @input="onTitleInput"
                        />
                    </div>
                    <div class="mb-2">
                        <label for="new-sp-slug" class="form-label">{{ $t("Slug") }}</label>
                        <div class="input-group">
                            <span class="input-group-text">/status/</span>
                            <input
                                id="new-sp-slug"
                                v-model="newSlug"
                                type="text"
                                class="form-control slug-input"
                                autocapitalize="none"
                                required
                                data-testid="slug-input"
                                @input="slugTouched = true"
                            />
                        </div>
                        <div class="form-text mt-2">
                            {{ $t("Accept characters:") }} <mark>a-z</mark> <mark>0-9</mark> <mark>-</mark>
                        </div>
                    </div>
                </div>
                <div class="modal-footer">
                    <button type="button" class="btn btn-normal" @click="hideAddDialog">
                        {{ $t("Cancel") }}
                    </button>
                    <button
                        type="submit"
                        class="btn btn-primary"
                        :disabled="processing"
                        data-testid="submit-button"
                    >
                        {{ $t("Next") }}
                    </button>
                </div>
            </form>
        </div>
    </div>
</template>

<script>
import { Modal } from "bootstrap";
import Confirm from "../components/Confirm.vue";
import { getResBaseURL } from "../util-frontend";

export default {
    components: {
        Confirm,
    },
    data() {
        return {
            selectedStatusSlug: "",
            newTitle: "",
            newSlug: "",
            slugTouched: false,
            processing: false,
            addModal: null,
        };
    },
    computed: {},
    mounted() {
        this.addModal = new Modal(this.$refs.addModal);
    },
    methods: {
        /**
         * Open the create-status-page dialog, resetting its fields.
         * @returns {void}
         */
        showAddDialog() {
            this.newTitle = "";
            this.newSlug = "";
            this.slugTouched = false;
            this.processing = false;
            this.addModal.show();
            this.$nextTick(() => this.$refs.nameInput?.focus());
        },
        hideAddDialog() {
            this.addModal.hide();
        },
        /**
         * Turn text into a valid slug: lowercase, a-z/0-9/dash only, no
         * leading/trailing or consecutive dashes.
         * @param {string} text Source text.
         * @returns {string} Slug.
         */
        slugify(text) {
            return String(text)
                .toLowerCase()
                .replace(/[^a-z0-9]+/g, "-")
                .replace(/-+/g, "-")
                .replace(/^-|-$/g, "");
        },
        /**
         * Auto-fill the slug from the name until the user edits the slug directly.
         * @returns {void}
         */
        onTitleInput() {
            if (!this.slugTouched) {
                this.newSlug = this.slugify(this.newTitle);
            }
        },
        /**
         * Create the status page and open it in the editor.
         * @returns {void}
         */
        createStatusPage() {
            this.processing = true;
            this.$root.getSocket().emit("addStatusPage", this.newTitle, this.newSlug, (res) => {
                this.processing = false;
                if (res.ok) {
                    location.href = "/status/" + res.slug + "?edit";
                } else if (res.msg && res.msg.includes("UNIQUE constraint")) {
                    this.$root.toastError("The slug is already taken. Please choose another slug.");
                } else {
                    this.$root.toastRes(res);
                }
            });
        },
        /**
         * Get the correct URL for the icon
         * @param {string} icon Path for icon
         * @returns {string} Correctly formatted path including port numbers
         */
        icon(icon) {
            if (icon === "/icon.svg") {
                return icon;
            } else {
                return getResBaseURL() + icon;
            }
        },
        deleteDialog(slug) {
            this.$data.selectedStatusSlug = slug;
            this.$refs.confirmDelete.show();
        },
        deleteStatusPage() {
            this.$root.getSocket().emit("deleteStatusPage", this.$data.selectedStatusSlug, (res) => {
                if (res.ok) {
                    this.$root.toastSuccess(this.$t("successDeleted"));
                    window.location.reload();
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

.slug-input {
    text-transform: lowercase;
}

.item {
    display: flex;
    align-items: center;
    gap: 10px;
    text-decoration: none;
    border-radius: 10px;
    transition: all ease-in-out 0.15s;
    padding: 10px;

    &:hover {
        background-color: $highlight-white;

        & .actions {
            visibility: visible;
        }
    }

    &.active {
        background-color: #cdf8f4;
    }

    $logo-width: 70px;

    .logo {
        width: $logo-width;
        height: $logo-width;

        // Better when the image is loading
        min-height: 1px;
    }

    .info {
        flex: 1 1 auto;

        .title {
            font-weight: bold;
            font-size: 20px;
        }

        .slug {
            font-size: 14px;
        }
    }

    .actions {
        visibility: hidden;
        display: flex;
        align-items: center;

        .delete-status-page {
            flex: 1 1 auto;
            display: inline-flex;
            align-items: center;
            gap: 0.25rem;
        }
    }
}

.dark {
    .item {
        &:hover {
            background-color: $dark-bg2;
        }

        &.active {
            background-color: $dark-bg2;
        }
    }
}

@media (max-width: 770px) {
    .item {
        .actions {
            visibility: visible;

            .btn {
                padding: 10px;
            }

            span {
                display: none;
            }
        }
    }
}
</style>
