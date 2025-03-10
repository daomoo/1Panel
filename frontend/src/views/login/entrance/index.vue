<template>
    <div>
        <div v-if="!loading">
            <div class="login-background" v-if="errStatus === ''">
                <div class="login-wrapper">
                    <div :class="screenWidth > 1110 ? 'left inline-block' : ''">
                        <div class="login-title">
                            <span>{{ globalStore.themeConfig.title || $t('setting.description') }}</span>
                        </div>
                        <img src="@/assets/images/1panel-login.png" alt="" v-if="screenWidth > 1110" />
                    </div>
                    <div :class="screenWidth > 1110 ? 'right inline-block' : ''">
                        <div class="login-container">
                            <LoginForm ref="loginRef"></LoginForm>
                        </div>
                    </div>
                </div>
            </div>

            <div v-else>
                <div v-if="errStatus === 'err-unsafe'">
                    <UnSafe />
                </div>
                <div v-if="errStatus === 'err-ip'">
                    <ErrIP />
                </div>
                <div v-if="errStatus === 'err-domain'">
                    <ErrDomain />
                </div>
                <div v-if="errStatus.indexOf('code-') !== -1">
                    <ErrCode :code="errStatus.replaceAll('code-', '')" />
                </div>
                <div v-if="errStatus === 'err-found'">
                    <ErrFound />
                </div>
            </div>
        </div>
    </div>
</template>

<script setup lang="ts" name="login">
import { checkIsSafety } from '@/api/modules/auth';
import LoginForm from '../components/login-form.vue';
import UnSafe from '@/components/error-message/unsafe.vue';
import ErrIP from '@/components/error-message/err_ip.vue';
import ErrCode from '@/components/error-message/error_code.vue';
import ErrDomain from '@/components/error-message/err_domain.vue';
import ErrFound from '@/components/error-message/404.vue';
import { ref, onMounted, watch } from 'vue';
import { GlobalStore } from '@/store';
import { getXpackSetting, initFavicon } from '@/utils/xpack';
const globalStore = GlobalStore();

const screenWidth = ref(null);
const errStatus = ref('');
const loading = ref();

const mySafetyCode = defineProps({
    code: {
        type: String,
        default: '',
    },
});
watch(
    () => globalStore.errStatus,
    (newVal) => {
        if (newVal?.startsWith('err-') || newVal?.startsWith('code-')) {
            errStatus.value = newVal;
        }
    },
);
const getStatus = async () => {
    let info = globalStore.errStatus;
    if (info?.startsWith('err-') || info?.startsWith('code-')) {
        errStatus.value = info;
        return;
    }
    let code = mySafetyCode.code;
    globalStore.entrance = code;
    loading.value = true;
    await checkIsSafety(code)
        .then(() => {
            loading.value = false;
            loadDataFromXDB();
        })
        .catch(() => {
            loading.value = false;
        });
};

const loadDataFromXDB = async () => {
    const res = await getXpackSetting();
    if (res) {
        globalStore.themeConfig.title = res.data.title;
        globalStore.themeConfig.logo = res.data.logo;
        globalStore.themeConfig.logoWithText = res.data.logoWithText;
        globalStore.themeConfig.favicon = res.data.favicon;
    }
    loading.value = false;
    initFavicon();
};

onMounted(() => {
    globalStore.isOnRestart = false;
    getStatus();
    screenWidth.value = document.body.clientWidth;
    window.onresize = () => {
        return (() => {
            screenWidth.value = document.body.clientWidth;
        })();
    };
});
</script>

<style scoped lang="scss">
@mixin login-center {
    display: flex;
    justify-content: center;
    align-items: center;
}

.login-background {
    height: 100vh;
    background: url(@/assets/images/1panel-login-bg.png) no-repeat,
        radial-gradient(153.25% 257.2% at 118.99% 181.67%, rgba(50, 132, 255, 0.2) 0%, rgba(82, 120, 255, 0) 100%)
            /* warning: gradient uses a rotation that is not supported by CSS and may not behave as expected */,
        radial-gradient(123.54% 204.83% at 25.87% 195.17%, rgba(111, 76, 253, 0.15) 0%, rgba(122, 76, 253, 0) 78.85%)
            /* warning: gradient uses a rotation that is not supported by CSS and may not behave as expected */,
        linear-gradient(0deg, rgba(0, 94, 235, 0.03), rgba(0, 94, 235, 0.03)),
        radial-gradient(109.58% 109.58% at 31.53% -36.58%, rgba(0, 94, 235, 0.3) 0%, rgba(0, 94, 235, 0) 100%)
            /* warning: gradient uses a rotation that is not supported by CSS and may not behave as expected */,
        rgba(0, 57, 142, 0.05);

    .login-wrapper {
        padding-top: 8%;
        width: 80%;
        margin: 0 auto;
        // @media only screen and (max-width: 1440px) {
        //     width: 100%;
        //     padding-top: 6%;
        // }
        .left {
            vertical-align: middle;
            text-align: right;
            width: 60%;
            img {
                object-fit: contain;
                width: 100%;
                @media only screen and (min-width: 1440px) {
                    width: 85%;
                }
            }
        }
        .right {
            vertical-align: middle;
            width: 40%;
        }
    }

    .login-title {
        text-align: right;
        margin-right: 10%;
        span:first-child {
            color: #005eeb;
            font-size: 40px;
            font-family: pingFangSC-Regular;
            font-weight: 600;
            @media only screen and (max-width: 768px) {
                font-size: 35px;
            }
        }
        @media only screen and (max-width: 1110px) {
            margin-bottom: 20px;
            font-size: 35px;
            text-align: center;
            margin-right: 0;
        }
    }
    .login-container {
        margin-top: 40px;
        padding: 40px 0;
        width: 390px;
        box-sizing: border-box;
        background-color: rgba(255, 255, 255, 0.55);
        border-radius: 4px;
        box-shadow: 2px 4px 22px rgba(0, 94, 235, 0.2);
        @media only screen and (max-width: 1440px) {
            margin-top: 60px;
        }
        @media only screen and (max-width: 1110px) {
            margin: 60px auto 0;
        }
        @media only screen and (max-width: 768px) {
            width: 100%;
        }
    }
}
</style>
