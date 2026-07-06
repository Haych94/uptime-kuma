<template>
    <div :class="classes">
        <div v-if="!$root.socket.connected && !$root.socket.firstConnect" class="lost-connection">
            <div class="container-fluid">
                {{ $root.connectionErrorMsg }}
                <div v-if="$root.showReverseProxyGuide">
                    {{ $t("Using a Reverse Proxy?") }}
                    <a href="https://github.com/louislam/uptime-kuma/wiki/Reverse-Proxy" target="_blank">
                        {{ $t("Check how to config it for WebSocket") }}
                    </a>
                </div>
            </div>
        </div>

        <!-- Sidebar (all screen sizes; collapses to an icon rail when narrow) -->
        <aside class="sidebar">
            <router-link to="/dashboard" class="sidebar-brand">
                <span class="brand-dot" />
                <span class="title">{{ $t("Uptime Kuma") }}</span>
            </router-link>

            <a
                v-if="hasNewVersion"
                target="_blank"
                href="https://github.com/louislam/uptime-kuma/releases"
                class="btn btn-primary update-btn"
            >
                <font-awesome-icon icon="arrow-alt-circle-up" />
                {{ $t("New Update") }}
            </a>

            <nav v-if="$root.loggedIn" class="sidebar-nav">
                <router-link
                    to="/dashboard"
                    class="side-link"
                    :class="{ active: $route.path.startsWith('/dashboard') }"
                    :title="$t('Dashboard')"
                >
                    <font-awesome-icon icon="tachometer-alt" fixed-width />
                    <span>{{ $t("Dashboard") }}</span>
                </router-link>
                <router-link
                    to="/manage-status-page"
                    class="side-link"
                    :class="{ active: $route.path.includes('status-page') }"
                    :title="$t('Status Pages')"
                >
                    <font-awesome-icon icon="stream" fixed-width />
                    <span>{{ $t("Status Pages") }}</span>
                </router-link>
                <router-link
                    to="/maintenance"
                    class="side-link"
                    :class="{ active: $route.path.includes('maintenance') }"
                    :title="$t('Maintenance')"
                >
                    <font-awesome-icon icon="wrench" fixed-width />
                    <span>{{ $t("Maintenance") }}</span>
                </router-link>
                <router-link
                    to="/settings/general"
                    class="side-link"
                    :class="{ active: $route.path.includes('settings') }"
                    :title="$t('Settings')"
                >
                    <font-awesome-icon icon="cog" fixed-width />
                    <span>{{ $t("Settings") }}</span>
                </router-link>
            </nav>

            <div v-if="$root.loggedIn" class="sidebar-footer">
                <div class="user-row">
                    <div class="profile-pic">{{ $root.usernameFirstChar }}</div>
                    <span class="username">{{ displayName }}</span>
                    <button
                        v-if="$root.socket.token !== 'autoLogin'"
                        class="logout-btn"
                        :title="$t('Logout')"
                        @click="$root.logout"
                    >
                        <font-awesome-icon icon="sign-out-alt" />
                    </button>
                </div>
            </div>
        </aside>

        <main class="has-sidebar">
            <router-view v-if="$root.loggedIn" />
            <Login v-if="!$root.loggedIn && $root.allowLoginDialog" />
        </main>

        <button
            v-if="numActiveToasts != 0"
            type="button"
            class="btn btn-normal clear-all-toast-btn"
            @click="clearToasts"
        >
            <font-awesome-icon icon="times" />
        </button>
    </div>
</template>

<script>
import Login from "../components/Login.vue";
import compareVersions from "compare-versions";
import { useToast } from "vue-toastification";
const toast = useToast();

export default {
    components: {
        Login,
    },

    data() {
        return {
            toastContainer: null,
            numActiveToasts: 0,
            toastContainerObserver: null,
        };
    },

    computed: {
        // Theme or Mobile
        classes() {
            const classes = {};
            classes[this.$root.theme] = true;
            classes["mobile"] = this.$root.isMobile;
            return classes;
        },

        hasNewVersion() {
            if (this.$root.info.latestVersion && this.$root.info.version) {
                return compareVersions(this.$root.info.latestVersion, this.$root.info.version) >= 1;
            } else {
                return false;
            }
        },

        /**
         * Friendly first name for the sidebar footer, derived from the
         * username: strips any email domain, takes the first word-ish
         * segment, drops trailing digits and capitalizes it.
         * e.g. "haych710@gmail.com" -> "Haych"
         * @returns {string} Display name, or the disabled-auth label.
         */
        displayName() {
            if (this.$root.username == null) {
                return this.$t("signedInDispDisabled");
            }
            const local = this.$root.username.split("@")[0];
            let name = local.split(/[\s._-]+/)[0];
            name = name.replace(/\d+$/, "") || local;
            return name.charAt(0).toUpperCase() + name.slice(1);
        },
    },

    watch: {},

    mounted() {
        this.toastContainer = document.querySelector(".bottom-right.toast-container");

        // Watch the number of active toasts
        this.toastContainerObserver = new MutationObserver((mutations) => {
            for (const mutation of mutations) {
                if (mutation.type === "childList") {
                    this.numActiveToasts = mutation.target.children.length;
                }
            }
        });

        if (this.toastContainer != null) {
            this.toastContainerObserver.observe(this.toastContainer, { childList: true });
        }
    },

    beforeUnmount() {
        this.toastContainerObserver.disconnect();
    },

    methods: {
        /**
         * Clear all toast notifications.
         * @returns {void}
         */
        clearToasts() {
            toast.clear();
        },
    },
};
</script>

<style lang="scss" scoped>
@import "../assets/vars.scss";

$sidebar-width: 268px;

// UptimeRobot-style dark left sidebar
.sidebar {
    position: fixed;
    top: 0;
    left: 0;
    width: $sidebar-width;
    height: 100vh;
    background-color: #171d28;
    display: flex;
    flex-direction: column;
    padding: 28px 16px 16px;
    z-index: 1000;
    overflow-y: auto;
}

.sidebar-brand {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 0 10px;
    margin-bottom: 34px;
    text-decoration: none;

    .brand-dot {
        width: 9px;
        height: 9px;
        border-radius: 50%;
        background-color: $primary;
        flex-shrink: 0;
    }

    .title {
        font-size: 21px;
        font-weight: 800;
        letter-spacing: -0.4px;
        color: #fff;
    }
}

.update-btn {
    margin-bottom: 12px;
}

.sidebar-nav {
    display: flex;
    flex-direction: column;
    gap: 8px;
}

// Small text, small icons, generous padding — tall airy rows like UptimeRobot
.side-link {
    display: flex;
    align-items: center;
    gap: 16px;
    padding: 15px 18px;
    border-radius: 10px;
    color: #97a0b0;
    text-decoration: none;
    font-weight: 500;
    font-size: 13px;
    line-height: 1.2;
    transition:
        background-color 0.15s,
        color 0.15s;

    svg {
        font-size: 15px;
        color: #77808f;
        transition: color 0.15s;
    }

    &:hover {
        background-color: rgba(9, 13, 20, 0.35);
        color: #dfe5ec;

        svg {
            color: #aeb7c5;
        }
    }

    // Active row is darker than the sidebar, like UptimeRobot
    &.router-link-exact-active,
    &.active {
        background-color: #10151d;
        color: #fff;

        svg {
            color: $primary;
        }
    }
}

.sidebar-footer {
    margin-top: auto;
    display: flex;
    flex-direction: column;
    gap: 4px;
    padding-top: 10px;
    border-top: 1px solid rgba(255, 255, 255, 0.07);

    .user-row {
        display: flex;
        align-items: center;
        gap: 10px;
        padding: 8px 10px;

        .profile-pic {
            width: 30px;
            height: 30px;
            border-radius: 50rem;
            background-color: $primary;
            color: #fff;
            display: flex;
            align-items: center;
            justify-content: center;
            font-weight: 700;
            font-size: 13px;
            flex-shrink: 0;
        }

        .username {
            color: #cdd4de;
            font-size: 14px;
            overflow: hidden;
            text-overflow: ellipsis;
            white-space: nowrap;
            flex: 1;
        }

        .logout-btn {
            background: none;
            border: none;
            color: #8b93a1;
            padding: 4px 8px;
            border-radius: 8px;
            cursor: pointer;

            &:hover {
                color: #fff;
                background-color: rgba(255, 255, 255, 0.08);
            }
        }
    }
}

main.has-sidebar {
    margin-left: $sidebar-width;
    min-height: 100vh;
    padding: 28px 20px;
}

// Collapse to an icon-only rail on narrow screens (UptimeRobot style).
// Applies at every width below 1100px, including phones.
$rail-width: 68px;

@media (max-width: 1100px) {
    .sidebar {
        width: $rail-width;
        padding: 20px 10px 14px;
        align-items: center;
    }

    .sidebar-brand {
        padding: 0;
        margin-bottom: 26px;

        .brand-dot {
            width: 26px;
            height: 26px;
        }

        .title {
            display: none;
        }
    }

    .update-btn {
        display: none;
    }

    .sidebar-nav {
        width: 100%;
    }

    .side-link {
        justify-content: center;
        gap: 0;
        padding: 15px 0;

        span {
            display: none;
        }

        svg {
            font-size: 17px;
        }
    }

    .sidebar-footer {
        width: 100%;

        .user-row {
            flex-direction: column;
            gap: 10px;
            padding: 10px 0 0;

            .profile-pic {
                width: 34px;
                height: 34px;
                font-size: 14px;
            }

            .username {
                display: none;
            }
        }
    }

    main.has-sidebar {
        margin-left: $rail-width;
    }
}

@media (max-width: 767.98px) {
    main.has-sidebar {
        padding: 18px 12px;
    }
}

.nav-link {
    &:hover {
        background-color: $primary;
        color: #fff;

        .dark & {
            background-color: $primary;
            color: #000;
        }

        &.active {
            background-color: $highlight;
        }
    }

    &.status-page {
        background-color: rgba(255, 255, 255, 0.1);
    }
}

main {
    min-height: calc(100vh - 160px);
}

.title {
    font-weight: bold;
}

.nav {
    margin-right: 25px;
}

.lost-connection {
    padding: 5px;
    background-color: crimson;
    color: white;
    position: fixed;
    width: 100%;
    z-index: 99999;
}

// Profile Pic Button with Dropdown
.dropdown-profile-pic {
    user-select: none;

    .nav-link {
        cursor: pointer;
        display: flex;
        gap: 6px;
        align-items: center;
        background-color: rgba(200, 200, 200, 0.2);
        padding: 0.5rem 0.8rem;

        &:hover {
            background-color: rgba(255, 255, 255, 0.2);
        }
    }

    .dropdown-menu {
        transition: all 0.2s;
        padding-left: 0;
        padding-bottom: 0;
        margin-top: 8px !important;
        border-radius: 16px;
        overflow: hidden;

        .dropdown-divider {
            margin: 0;
            border-top: 1px solid rgba(0, 0, 0, 0.4);
            background-color: transparent;
        }

        .dropdown-item-text {
            font-size: 14px;
            padding-bottom: 0.7rem;
        }

        .dropdown-item {
            padding: 0.7rem 1rem;
        }

        .dark & {
            background-color: $dark-bg;
            color: $dark-font-color;
            border-color: $dark-border-color;

            .dropdown-item {
                color: $dark-font-color;

                &.active {
                    color: $dark-font-color2;
                    background-color: $highlight !important;
                }

                &:hover {
                    background-color: $dark-bg2;
                }
            }
        }
    }

    .profile-pic {
        display: flex;
        align-items: center;
        justify-content: center;
        color: white;
        background-color: $primary;
        width: 24px;
        height: 24px;
        margin-right: 5px;
        border-radius: 50rem;
        font-weight: bold;
        font-size: 10px;
    }
}


.clear-all-toast-btn {
    position: fixed;
    right: 1em;
    bottom: 1em;
    font-size: 1.2em;
    padding: 9px 15px;
    width: 48px;
    box-shadow: 2px 2px 30px rgba(0, 0, 0, 0.2);
    z-index: 100;

    .dark & {
        box-shadow: 2px 2px 30px rgba(0, 0, 0, 0.5);
    }
}

@media (max-width: 770px) {
    .clear-all-toast-btn {
        bottom: 72px;
    }
}
</style>
