<template>
    <transition ref="tableContainer" name="slide-fade" appear>
        <div v-if="$route.name === 'DashboardHome'">
            <div class="d-flex align-items-center justify-content-between mb-4">
                <h1 class="mb-0">
                    {{ $t("monitorsHeading") }}
                </h1>
                <router-link to="/add" class="btn btn-blue fw-normal">
                    <font-awesome-icon icon="plus" />
                    <span class="ms-1">{{ $t("New") }}</span>
                </router-link>
            </div>

            <div class="row g-4 align-items-start">
                <div class="col-12 col-xl-8 col-xxl-9">
                    <MonitorList />

                    <div class="shadow-box table-shadow-box table-wrapper mt-4">
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

                <div class="col-12 col-xl-4 col-xxl-3 side-rail">
                    <div class="shadow-box side-card big-padding">
                        <h2 class="side-title">{{ $t("Current status") }}</h2>
                        <div class="status-orb" :class="$root.stats.down > 0 ? 'bad' : 'ok'">
                            <font-awesome-icon
                                :icon="$root.stats.down > 0 ? 'exclamation-circle' : 'arrow-alt-circle-up'"
                            />
                        </div>
                        <div class="status-counts">
                            <div class="count">
                                <span class="num" :class="{ 'text-danger': $root.stats.down > 0 }">
                                    {{ $root.stats.down }}
                                </span>
                                <span class="lbl">{{ $t("Down") }}</span>
                            </div>
                            <div class="count">
                                <span class="num">{{ $root.stats.up }}</span>
                                <span class="lbl">{{ $t("Up") }}</span>
                            </div>
                            <div class="count">
                                <span class="num">{{ $root.stats.pause }}</span>
                                <span class="lbl">{{ $t("filterActivePaused") }}</span>
                            </div>
                        </div>
                        <div v-if="$root.stats.maintenance > 0 || $root.stats.unknown > 0" class="status-counts secondary">
                            <div v-if="$root.stats.maintenance > 0" class="count">
                                <span class="num text-maintenance">{{ $root.stats.maintenance }}</span>
                                <span class="lbl">{{ $t("Maintenance") }}</span>
                            </div>
                            <div v-if="$root.stats.unknown > 0" class="count">
                                <span class="num">{{ $root.stats.unknown }}</span>
                                <span class="lbl">{{ $t("Unknown") }}</span>
                            </div>
                        </div>
                        <p class="side-caption">{{ $t("monitorsTotal", { count: totalMonitors }) }}</p>
                    </div>

                    <div class="shadow-box side-card big-padding mt-4">
                        <h2 class="side-title">{{ $t("Last 24 hours") }}</h2>
                        <div class="pair-grid">
                            <div class="pair">
                                <span class="big text-primary-c">{{ overallUptimeDisplay }}</span>
                                <span class="lbl">{{ $t("Overall uptime") }}</span>
                            </div>
                            <div class="pair">
                                <span class="big">{{ incidents24h }}</span>
                                <span class="lbl">{{ $t("Incidents") }}</span>
                            </div>
                        </div>
                    </div>
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
import MonitorList from "../components/MonitorList.vue";
import { DOWN } from "../util.ts";

export default {
    components: {
        Datetime,
        Status,
        Pagination,
        Confirm,
        MonitorList,
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

        totalMonitors() {
            const s = this.$root.stats;
            return (s.up || 0) + (s.down || 0) + (s.maintenance || 0) + (s.unknown || 0) + (s.pause || 0);
        },

        /**
         * Average of every monitor's 24-hour uptime, as a display string.
         * @returns {string} e.g. "99.98%", or "N/A" before any data arrives.
         */
        overallUptimeDisplay() {
            const values = Object.entries(this.$root.uptimeList)
                .filter(([key]) => key.endsWith("_24"))
                .map(([, value]) => value);
            if (values.length === 0) {
                return "N/A";
            }
            const avg = (values.reduce((sum, v) => sum + v, 0) / values.length) * 100;
            return `${parseFloat(avg.toFixed(2))}%`;
        },

        /**
         * Number of loaded important events that are DOWN beats from the
         * last 24 hours. Based on the currently loaded page of events.
         * @returns {number} Count of down events in the last 24 hours.
         */
        incidents24h() {
            const dayAgo = Date.now() - 24 * 60 * 60 * 1000;
            return this.displayedRecords.filter((beat) => {
                if (beat.status !== DOWN) {
                    return false;
                }
                const t = Date.parse(String(beat.time).replace(" ", "T"));
                return !isNaN(t) && t >= dayAgo;
            }).length;
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

// Right rail — UptimeRobot-style summary cards
.side-card {
    .side-title {
        font-size: 17px;
        margin-bottom: 18px;
    }

    .status-orb {
        width: 56px;
        height: 56px;
        border-radius: 50%;
        display: flex;
        align-items: center;
        justify-content: center;
        font-size: 24px;
        margin: 6px auto 18px;

        &.ok {
            background: rgba(59, 214, 113, 0.15);
            color: $primary;
        }

        &.bad {
            background: rgba(220, 53, 69, 0.15);
            color: $danger;
        }
    }

    .status-counts {
        display: grid;
        grid-template-columns: repeat(3, minmax(0, 1fr));
        text-align: center;
        gap: 8px;

        &.secondary {
            margin-top: 12px;
            grid-template-columns: repeat(2, minmax(0, 1fr));
        }

        .count {
            display: flex;
            flex-direction: column;
            line-height: 1.2;
        }

        .num {
            font-size: 22px;
            font-weight: 700;

            .dark & {
                color: #fff;
            }

            &.text-danger {
                color: $danger !important;
            }
        }

        .lbl {
            font-size: 12.5px;
            color: $secondary-text;
            margin-top: 2px;
        }
    }

    .side-caption {
        text-align: center;
        color: $secondary-text;
        font-size: 13px;
        margin: 16px 0 0;
    }

    .pair-grid {
        display: grid;
        grid-template-columns: repeat(2, minmax(0, 1fr));
        gap: 14px;

        .pair {
            display: flex;
            flex-direction: column;
            line-height: 1.25;
        }

        .big {
            font-size: 22px;
            font-weight: 700;

            .dark & {
                color: #fff;
            }

            &.text-primary-c {
                color: $primary;

                .dark & {
                    color: $primary;
                }
            }
        }

        .lbl {
            font-size: 12.5px;
            color: $secondary-text;
            margin-top: 2px;
        }
    }
}

.shadow-box:not(.side-card) {
    padding: 20px;
}

table {
    font-size: 14px;
    border-collapse: separate;
    border-spacing: 0;

    tr {
        transition: background-color 0.15s ease;
    }

    td {
        padding-top: 12px;
        padding-bottom: 12px;
        vertical-align: middle;
    }

    // Soft, rounded row hover to match the monitor list rows
    &.table-hover > tbody > tr:hover > td {
        --bs-table-accent-bg: transparent;
        background-color: rgba(255, 255, 255, 0.04);

        &:first-child {
            border-top-left-radius: 10px;
            border-bottom-left-radius: 10px;
        }

        &:last-child {
            border-top-right-radius: 10px;
            border-bottom-right-radius: 10px;
        }
    }

    body:not(.dark) &.table-hover > tbody > tr:hover > td {
        background-color: rgba(0, 0, 0, 0.035);
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
