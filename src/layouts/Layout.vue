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

        <!-- Desktop sidebar -->
        <aside v-if="!$root.isMobile" class="sidebar">
            <router-link to="/dashboard" class="sidebar-brand">
                <object class="brand-icon" width="32" height="32" data="/icon.svg" />
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
                >
                    <font-awesome-icon icon="tachometer-alt" fixed-width />
                    <span>{{ $t("Dashboard") }}</span>
                </router-link>
                <router-link
                    to="/manage-status-page"
                    class="side-link"
                    :class="{ active: $route.path.includes('status-page') }"
                >
                    <font-awesome-icon icon="stream" fixed-width />
                    <span>{{ $t("Status Pages") }}</span>
                </router-link>
                <router-link
                    to="/maintenance"
                    class="side-link"
                    :class="{ active: $route.path.includes('maintenance') }"
                >
                    <font-awesome-icon icon="wrench" fixed-width />
                    <span>{{ $t("Maintenance") }}</span>
                </router-link>
                <router-link
                    to="/settings/general"
                    class="side-link"
                    :class="{ active: $route.path.includes('settings') }"
                >
                    <font-awesome-icon icon="cog" fixed-width />
                    <span>{{ $t("Settings") }}</span>
                </router-link>
            </nav>

            <div v-if="$root.loggedIn" class="sidebar-footer">
                <a href="https://github.com/louislam/uptime-kuma/wiki" target="_blank" class="side-link">
                    <font-awesome-icon icon="info-circle" fixed-width />
                    <span>{{ $t("Help") }}</span>
                </a>
                <div class="user-row">
                    <div class="profile-pic">{{ $root.usernameFirstChar }}</div>
                    <span class="username">{{ $root.username == null ? $t("signedInDispDisabled") : $root.username }}</span>
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

        <!-- Mobile header -->
        <header v-else class="d-flex flex-wrap justify-content-center pt-2 pb-2 mb-3">
            <router-link to="/dashboard" class="d-flex align-items-center text-dark text-decoration-none">
                <object class="bi" width="40" height="40" data="/icon.svg" />
                <span class="fs-4 title ms-2">Uptime Kuma</span>
            </router-link>
        </header>

        <main :class="{ 'has-sidebar': !$root.isMobile }">
            <router-view v-if="$root.loggedIn" />
            <Login v-if="!$root.loggedIn && $root.allowLoginDialog" />
        </main>

        <!-- Mobile Only -->
        <div v-if="$root.isMobile" style="width: 100%; height: calc(60px + env(safe-area-inset-bottom))" />
        <nav v-if="$root.isMobile && $root.loggedIn" class="bottom-nav">
            <router-link to="/dashboard" class="nav-link">
                <div><font-awesome-icon icon="tachometer-alt" /></div>
                {{ $t("Home") }}
            </router-link>

            <router-link to="/list" class="nav-link">
                <div><font-awesome-icon icon="list" /></div>
                {{ $t("List") }}
            </router-link>

            <router-link to="/add" class="nav-link">
                <div><font-awesome-icon icon="plus" /></div>
                {{ $t("Add") }}
            </router-link>

            <router-link to="/settings" class="nav-link">
                <div><font-awesome-icon icon="cog" /></div>
                {{ $t("Settings") }}
            </router-link>
        </nav>

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

$sidebar-width: 244px;

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
    padding: 20px 14px;
    z-index: 1000;
    overflow-y: auto;
}

.sidebar-brand {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 6px 8px 18px;
    text-decoration: none;

    .title {
        font-size: 20px;
        font-weight: 700;
        color: #fff;

        &::after {
            content: ".";
            color: $primary;
        }
    }
}

.update-btn {
    margin-bottom: 12px;
}

.sidebar-nav {
    display: flex;
    flex-direction: column;
    gap: 4px;
}

.side-link {
    display: flex;
    align-items: center;
    gap: 14px;
    padding: 12px 14px;
    border-radius: 10px;
    color: #97a0b0;
    text-decoration: none;
    font-weight: 500;
    font-size: 15px;
    transition:
        background-color 0.15s,
        color 0.15s;

    svg {
        font-size: 17px;
        color: #8b93a1;
        transition: color 0.15s;
    }

    &:hover {
        background-color: rgba(255, 255, 255, 0.05);
        color: #fff;

        svg {
            color: #cdd4de;
        }
    }

    &.router-link-exact-active,
    &.active {
        background-color: rgba(255, 255, 255, 0.07);
        color: #fff;
        font-weight: 600;

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

.bottom-nav {
    z-index: 1000;
    position: fixed;
    bottom: 0;
    height: calc(60px + env(safe-area-inset-bottom));
    width: 100%;
    left: 0;
    background-color: #fff;
    box-shadow:
        0 15px 47px 0 rgba(0, 0, 0, 0.05),
        0 5px 14px 0 rgba(0, 0, 0, 0.05);
    text-align: center;
    white-space: nowrap;
    padding: 0 10px env(safe-area-inset-bottom);

    a {
        text-align: center;
        width: 25%;
        display: inline-block;
        height: 100%;
        padding: 8px 10px 0;
        font-size: 13px;
        color: #c1c1c1;
        overflow: hidden;
        text-decoration: none;

        &.router-link-exact-active,
        &.active {
            color: $primary;
            font-weight: bold;
        }

        div {
            font-size: 20px;
        }
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

.dark {
    header {
        background-color: $dark-header-bg;
        border-bottom-color: $dark-header-bg !important;

        span {
            color: #f0f6fc;
        }
    }

    .bottom-nav {
        background-color: $dark-bg;
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
