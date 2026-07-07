<template>
    <transition name="slide-fade" appear>
        <div>
            <div class="mb-4">
                <h1 class="mb-1">{{ $t("Incidents") }}</h1>
                <p class="subtitle mb-0">{{ $t("incidentsSubtitle") }}</p>
            </div>

            <div class="shadow-box table-wrapper">
                <table class="table table-borderless table-hover">
                    <thead>
                        <tr>
                            <th>{{ $t("Status") }}</th>
                            <th class="monitor-col">{{ $t("Monitor") }}</th>
                            <th>{{ $t("Root Cause") }}</th>
                            <th>{{ $t("Started") }}</th>
                            <th>{{ $t("Resolved") }}</th>
                            <th class="text-end">{{ $t("Duration") }}</th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr v-for="(incident, index) in incidents" :key="index">
                            <td>
                                <span class="incident-badge" :class="incident.resolved ? 'resolved' : 'ongoing'">
                                    {{ incident.resolved ? $t("Resolved") : $t("Ongoing") }}
                                </span>
                            </td>
                            <td class="monitor-col">
                                <router-link v-if="monitorName(incident.monitorID)" :to="`/dashboard/${incident.monitorID}`">
                                    {{ monitorName(incident.monitorID) }}
                                </router-link>
                                <span v-else class="text-secondary">#{{ incident.monitorID }}</span>
                            </td>
                            <td class="root-cause">{{ incident.rootCause || "—" }}</td>
                            <td><Datetime :value="incident.start" /></td>
                            <td>
                                <Datetime v-if="incident.resolved" :value="incident.resolved" />
                                <span v-else class="text-secondary">—</span>
                            </td>
                            <td class="text-end duration">{{ durationText(incident) }}</td>
                        </tr>

                        <tr v-if="incidents.length === 0">
                            <td colspan="6" class="text-center text-secondary py-4">
                                {{ $t("No incidents") }}
                            </td>
                        </tr>
                    </tbody>
                </table>
            </div>
        </div>
    </transition>
</template>

<script>
import Datetime from "../components/Datetime.vue";
import { UP, DOWN } from "../util.ts";

export default {
    components: {
        Datetime,
    },
    data() {
        return {
            beats: [],
        };
    },
    computed: {
        /**
         * Build the incident list by pairing each DOWN status-change event with
         * its following recovery (UP) event, per monitor. An unresolved DOWN is
         * an ongoing incident. Sorted newest-first.
         * @returns {Array<object>} Incident objects.
         */
        incidents() {
            const byMonitor = {};
            for (const beat of this.beats) {
                (byMonitor[beat.monitorID] = byMonitor[beat.monitorID] || []).push(beat);
            }

            const result = [];
            for (const [monitorID, list] of Object.entries(byMonitor)) {
                list.sort((a, b) => this.toMs(a.time) - this.toMs(b.time));

                let open = null;
                for (const beat of list) {
                    if (beat.status === DOWN) {
                        if (!open) {
                            open = { monitorID: Number(monitorID), start: beat.time, rootCause: beat.msg, resolved: null };
                        }
                    } else if (beat.status === UP && open) {
                        open.resolved = beat.time;
                        result.push(open);
                        open = null;
                    }
                }
                if (open) {
                    result.push(open);
                }
            }

            result.sort((a, b) => this.toMs(b.start) - this.toMs(a.start));
            return result;
        },
    },
    mounted() {
        this.loadBeats();
        this.$root.emitter.on("newImportantHeartbeat", this.onNewImportantHeartbeat);
    },
    beforeUnmount() {
        this.$root.emitter.off("newImportantHeartbeat", this.onNewImportantHeartbeat);
    },
    methods: {
        /**
         * Fetch the most recent important (status-change) heartbeats across all monitors.
         * @returns {void}
         */
        loadBeats() {
            this.$root.getSocket().emit("monitorImportantHeartbeatListPaged", null, 0, 500, (res) => {
                if (res.ok) {
                    this.beats = res.data;
                }
            });
        },
        onNewImportantHeartbeat() {
            this.loadBeats();
        },
        /**
         * Parse a UTC heartbeat time string to epoch milliseconds.
         * @param {string} time Heartbeat time (UTC, "YYYY-MM-DD HH:mm:ss").
         * @returns {number} Epoch milliseconds.
         */
        toMs(time) {
            return new Date(String(time).replace(" ", "T") + "Z").getTime();
        },
        monitorName(monitorID) {
            return this.$root.monitorList[monitorID]?.name;
        },
        /**
         * Human-readable duration for an incident (to resolution, or to now if ongoing).
         * @param {object} incident Incident object.
         * @returns {string} e.g. "1d 3h", "2h 15m", "45s".
         */
        durationText(incident) {
            const end = incident.resolved ? this.toMs(incident.resolved) : Date.now();
            let seconds = Math.max(0, Math.floor((end - this.toMs(incident.start)) / 1000));

            const days = Math.floor(seconds / 86400);
            const hours = Math.floor((seconds % 86400) / 3600);
            const minutes = Math.floor((seconds % 3600) / 60);
            seconds = seconds % 60;

            if (days > 0) {
                return `${days}d ${hours}h`;
            }
            if (hours > 0) {
                return `${hours}h ${minutes}m`;
            }
            if (minutes > 0) {
                return `${minutes}m ${seconds}s`;
            }
            return `${seconds}s`;
        },
    },
};
</script>

<style lang="scss" scoped>
@import "../assets/vars.scss";

.subtitle {
    color: $secondary-text;
    font-size: 14px;
}

.shadow-box {
    padding: 20px;
}

.table-wrapper {
    overflow-x: auto;
}

table {
    font-size: 14px;
    margin-bottom: 0;
    border-collapse: separate;
    border-spacing: 0;

    td {
        padding-top: 14px;
        padding-bottom: 14px;
        vertical-align: middle;
    }

    // Soft, rounded row hover to match the monitor list rows
    tbody tr {
        transition: background-color 0.15s ease;
    }

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

    .monitor-col {
        min-width: 160px;
    }

    .root-cause {
        color: $secondary-text;
        max-width: 320px;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }

    .duration {
        font-variant-numeric: tabular-nums;
        white-space: nowrap;
    }
}

.incident-badge {
    display: inline-block;
    font-size: 11px;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.4px;
    padding: 3px 10px;
    border-radius: 999px;
    white-space: nowrap;

    &.ongoing {
        color: #fff;
        background-color: $danger;
    }

    &.resolved {
        color: $primary;
        background-color: rgba(59, 214, 113, 0.15);
    }
}
</style>
