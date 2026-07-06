<template>
    <transition ref="tableContainer" name="slide-fade" appear>
        <div v-if="$route.name === 'DashboardHome'">
            <div class="d-flex align-items-center justify-content-between mb-3">
                <h1 class="mb-0">
                    {{ $t("Quick Stats") }}
                </h1>
                <router-link to="/add" class="btn btn-primary">
                    <font-awesome-icon icon="plus" />
                    {{ $t("Add New Monitor") }}
                </router-link>
            </div>

            <div class="stat-tiles mb-4">
                <div class="stat-tile shadow-box" :class="{ active: $root.stats.up > 0 }">
                    <div class="icon up"><font-awesome-icon icon="heartbeat" /></div>
                    <div class="tile-body">
                        <span class="tile-num" :class="$root.stats.up === 0 && 'muted'">{{ $root.stats.up }}</span>
                        <span class="tile-label">{{ $t("Up") }}</span>
                    </div>
                </div>
                <div class="stat-tile shadow-box" :class="{ active: $root.stats.down > 0 }">
                    <div class="icon down"><font-awesome-icon icon="times-circle" /></div>
                    <div class="tile-body">
                        <span class="tile-num" :class="$root.stats.down > 0 ? 'text-danger' : 'muted'">{{ $root.stats.down }}</span>
                        <span class="tile-label">{{ $t("Down") }}</span>
                    </div>
                </div>
                <div class="stat-tile shadow-box" :class="{ active: $root.stats.maintenance > 0 }">
                    <div class="icon maintenance"><font-awesome-icon icon="wrench" /></div>
                    <div class="tile-body">
                        <span class="tile-num" :class="$root.stats.maintenance > 0 ? 'text-maintenance' : 'muted'">{{ $root.stats.maintenance }}</span>
                        <span class="tile-label">{{ $t("Maintenance") }}</span>
                    </div>
                </div>
                <div class="stat-tile shadow-box">
                    <div class="icon unknown"><font-awesome-icon icon="question-circle" /></div>
                    <div class="tile-body">
                        <span class="tile-num muted">{{ $root.stats.unknown }}</span>
                        <span class="tile-label">{{ $t("Unknown") }}</span>
                    </div>
                </div>
                <div class="stat-tile shadow-box">
                    <div class="icon pause"><font-awesome-icon icon="pause" /></div>
                    <div class="tile-body">
                        <span class="tile-num muted">{{ $root.stats.pause }}</span>
                        <span class="tile-label">{{ $t("pauseDashboardHome") }}</span>
                    </div>
                </div>
            </div>

            <div class="shadow-box table-shadow-box table-wrapper">
                <div class="d-flex align-items-center justify-content-between mb-3">
                    <h2 class="mb-0 events-title">{{ $t("Events") }}</h2>
                    <button
                        class="btn btn-sm btn-outline-danger"
                        :disabled="clearingAllEvents"
                        @click="clearAllEventsDialog"
                    >
                        {{ $t("Clear All Events") }}
                    </button>
                </div>
                <table class="table table-borderless table-hover">
                    <thead>
                        <tr>
                            <th v-if="showGroupColumn">{{ $t("Group Name") }}</th>
                            <th class="name-column">{{ $t("Name") }}</th>
                            <th>{{ $t("Status") }}</th>
                            <th>{{ $t("DateTime") }}</th>
                            <th>{{ $t("Message") }}</th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr
                            v-for="(beat, index) in displayedRecords"
                            :key="index"
                            :class="{ 'shadow-box': $root.windowWidth <= 550 }"
                        >
                            <td v-if="showGroupColumn">
                                <router-link
                                    v-if="getGroupName(beat.monitorID)"
                                    :to="`/dashboard/${getGroupId(beat.monitorID)}`"
                                >
                                    {{ getGroupName(beat.monitorID) }}
                                </router-link>
                                <span v-else class="text-secondary">—</span>
                            </td>
                            <td class="name-column">
                                <router-link :to="`/dashboard/${beat.monitorID}`">
                                    {{ $root.monitorList[beat.monitorID]?.name }}
                                </router-link>
                            </td>
                            <td><Status :status="beat.status" /></td>
                            <td :class="{ 'border-0': !beat.msg }"><Datetime :value="beat.time" /></td>
                            <td class="border-0">{{ beat.msg }}</td>
                        </tr>

                        <tr v-if="importantHeartBeatListLength === 0">
                            <td :colspan="tableColumnCount">
                                {{ $t("No important events") }}
                            </td>
                        </tr>
                    </tbody>
                </table>

                <div class="d-flex justify-content-center kuma_pagination">
                    <pagination
                        v-model="page"
                        :records="importantHeartBeatListLength"
                        :per-page="perPage"
                        :options="paginationConfig"
                    />
                </div>
            </div>
        </div>
    </transition>
    <Confirm
        ref="confirmClearEvents"
        btn-style="btn-danger"
        :yes-text="$t('Yes')"
        :no-text="$t('No')"
        @yes="clearAllEvents"
    >
        {{ $t("clearAllEventsMsg") }}
    </Confirm>
    <router-view ref="child" />
</template>

<script>
import Status from "../components/Status.vue";
import Datetime from "../components/Datetime.vue";
import Pagination from "v-pagination-3";
import Confirm from "../components/Confirm.vue";

export default {
    components: {
        Datetime,
        Status,
        Pagination,
        Confirm,
    },
    props: {
        calculatedHeight: {
            type: Number,
            default: 0,
        },
    },
    data() {
        return {
            page: 1,
            perPage: 25,
            initialPerPage: 25,
            paginationConfig: {
                hideCount: true,
                chunksNavigation: "scroll",
            },
            importantHeartBeatListLength: 0,
            displayedRecords: [],
            clearingAllEvents: false,
        };
    },
    computed: {
        showGroupColumn() {
            return Object.values(this.$root.monitorList).some((m) => m.parent != null);
        },
        tableColumnCount() {
            return this.showGroupColumn ? 5 : 4;
        },
    },
    watch: {
        perPage() {
            this.$nextTick(() => {
                this.getImportantHeartbeatListPaged();
            });
        },

        page() {
            this.getImportantHeartbeatListPaged();
        },
    },

    mounted() {
        this.getImportantHeartbeatListLength();

        this.$root.emitter.on("newImportantHeartbeat", this.onNewImportantHeartbeat);

        this.initialPerPage = this.perPage;

        window.addEventListener("resize", this.updatePerPage);
        this.updatePerPage();
    },

    beforeUnmount() {
        this.$root.emitter.off("newImportantHeartbeat", this.onNewImportantHeartbeat);

        window.removeEventListener("resize", this.updatePerPage);
    },

    methods: {
        /**
         * Returns the group (parent) name for a monitor, or empty string if none.
         * @param {number} monitorID - The monitor ID.
         * @returns {string} The group name or empty string.
         */
        getGroupName(monitorID) {
            const monitor = this.$root.monitorList[monitorID];
            if (!monitor || monitor.parent == null) {
                return "";
            }
            const parent = this.$root.monitorList[monitor.parent];
            return parent ? parent.name : "";
        },

        /**
         * Returns the group (parent) ID for a monitor, or null if none.
         * @param {number} monitorID - The monitor ID.
         * @returns {number|null} The group monitor ID or null.
         */
        getGroupId(monitorID) {
            const monitor = this.$root.monitorList[monitorID];
            return monitor && monitor.parent != null ? monitor.parent : null;
        },

        /**
         * Updates the displayed records when a new important heartbeat arrives.
         * @param {object} heartbeat - The heartbeat object received.
         * @returns {void}
         */
        onNewImportantHeartbeat(heartbeat) {
            if (this.page === 1) {
                this.displayedRecords.unshift(heartbeat);
                if (this.displayedRecords.length > this.perPage) {
                    this.displayedRecords.pop();
                }
                this.importantHeartBeatListLength += 1;
            }
        },

        /**
         * Retrieves the length of the important heartbeat list for all monitors.
         * @returns {void}
         */
        getImportantHeartbeatListLength() {
            this.$root.getSocket().emit("monitorImportantHeartbeatListCount", null, (res) => {
                if (res.ok) {
                    this.importantHeartBeatListLength = res.count;
                    this.getImportantHeartbeatListPaged();
                }
            });
        },

        /**
         * Retrieves the important heartbeat list for the current page.
         * @returns {void}
         */
        getImportantHeartbeatListPaged() {
            const offset = (this.page - 1) * this.perPage;
            this.$root.getSocket().emit("monitorImportantHeartbeatListPaged", null, offset, this.perPage, (res) => {
                if (res.ok) {
                    this.displayedRecords = res.data;
                }
            });
        },

        /**
         * Updates the number of items shown per page based on the available height.
         * @returns {void}
         */
        updatePerPage() {
            const tableContainer = this.$refs.tableContainer;
            const tableContainerHeight = tableContainer.offsetHeight;
            const availableHeight = window.innerHeight - tableContainerHeight;
            const additionalPerPage = Math.floor(availableHeight / 58);

            if (additionalPerPage > 0) {
                this.perPage = Math.max(this.initialPerPage, this.perPage + additionalPerPage);
            } else {
                this.perPage = this.initialPerPage;
            }
        },

        clearAllEventsDialog() {
            this.$refs.confirmClearEvents.show();
        },
        clearAllEvents() {
            this.clearingAllEvents = true;
            const monitorIDs = Object.keys(this.$root.monitorList);
            let failed = 0;
            const total = monitorIDs.length;

            if (total === 0) {
                this.clearingAllEvents = false;
                this.$root.toastError(this.$t("No monitors found"));
                return;
            }

            monitorIDs.forEach((monitorID) => {
                this.$root.getSocket().emit("clearEvents", monitorID, (res) => {
                    if (!res || !res.ok) {
                        failed++;
                    }
                });
            });
            this.clearingAllEvents = false;
            this.page = 1;
            this.getImportantHeartbeatListLength();
            if (failed === 0) {
                this.$root.toastSuccess(this.$t("Events cleared successfully"));
            } else {
                this.$root.toastError(
                    this.$t("Could not clear events", {
                        failed,
                        total,
                    })
                );
            }
        },
    },
};
</script>

<style lang="scss" scoped>
@import "../assets/vars";

.stat-tiles {
    display: grid;
    grid-template-columns: repeat(5, minmax(0, 1fr));
    gap: 16px;

    @media (max-width: 1100px) {
        grid-template-columns: repeat(3, minmax(0, 1fr));
    }

    @media (max-width: 600px) {
        grid-template-columns: repeat(2, minmax(0, 1fr));
        gap: 10px;
    }
}

.stat-tile {
    display: flex;
    align-items: center;
    gap: 14px;
    padding: 18px 20px;
    min-width: 0;

    @media (max-width: 600px) {
        padding: 14px;
        gap: 10px;

        .icon {
            width: 38px;
            height: 38px;
            font-size: 16px;
        }
    }
    transition: transform 0.15s ease, box-shadow 0.15s ease;

    &:hover {
        transform: translateY(-2px);
    }

    .icon {
        flex-shrink: 0;
        width: 44px;
        height: 44px;
        border-radius: 12px;
        display: flex;
        align-items: center;
        justify-content: center;
        font-size: 18px;

        &.up {
            color: $primary;
            background: rgba(59, 214, 113, 0.13);
        }
        &.down {
            color: $danger;
            background: rgba(220, 53, 69, 0.13);
        }
        &.maintenance {
            color: $maintenance;
            background: rgba(23, 71, 245, 0.12);
        }
        &.unknown,
        &.pause {
            color: $secondary-text;
            background: rgba(139, 147, 161, 0.15);
        }
    }

    .tile-body {
        display: flex;
        flex-direction: column;
        line-height: 1.1;
    }

    .tile-num {
        font-size: 26px;
        font-weight: 700;

        &.muted {
            color: $secondary-text;
        }
    }

    .tile-label {
        font-size: 13px;
        color: $secondary-text;
        margin-top: 2px;
    }
}

.shadow-box:not(.stat-tile) {
    padding: 20px;
}

table {
    font-size: 14px;

    tr {
        transition: all ease-in-out 0.2ms;
    }

    td {
        padding-top: 12px;
        padding-bottom: 12px;
        vertical-align: middle;
    }

    @media (max-width: 550px) {
        table-layout: fixed;
        overflow-wrap: break-word;
    }
}

@media screen and (max-width: 1280px) {
    .name-column {
        min-width: 150px;
    }
}

@media screen and (min-aspect-ratio: 4/3) {
    .name-column {
        min-width: 200px;
    }
}

.table-wrapper {
    overflow-x: auto;
}
</style>
