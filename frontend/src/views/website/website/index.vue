<template>
    <div>
        <RouterButton
            :buttons="[
                {
                    label: i18n.global.t('website.website'),
                    path: '/websites',
                },
            ]"
        />
        <LayoutContent :title="$t('website.website')" v-loading="loading">
            <template #app>
                <AppStatus
                    :app-key="'openresty'"
                    @setting="setting"
                    v-model:mask-show="maskShow"
                    v-model:loading="loading"
                    @is-exist="checkExist"
                ></AppStatus>
            </template>
            <template v-if="nginxIsExist && !openNginxConfig" #toolbar>
                <el-row :class="{ mask: nginxStatus != 'Running' }">
                    <el-col :xs="24" :sm="20" :md="20" :lg="20" :xl="20">
                        <el-button type="primary" @click="openCreate">
                            {{ $t('website.create') }}
                        </el-button>
                        <el-button type="primary" plain @click="openGroup">
                            {{ $t('website.group') }}
                        </el-button>
                        <el-button type="primary" plain @click="openDefault">
                            {{ $t('website.defaultServer') }}
                        </el-button>
                    </el-col>
                    <el-col :xs="24" :sm="4" :md="4" :lg="4" :xl="4">
                        <TableSearch @search="search()" v-model:searchName="req.name" />
                    </el-col>
                </el-row>
            </template>
            <template v-if="nginxIsExist && !openNginxConfig" #search>
                <div :class="{ mask: nginxStatus != 'Running' }">
                    <el-select v-model="req.websiteGroupId" @change="search()" class="p-w-200">
                        <template #prefix>{{ $t('website.group') }}</template>
                        <el-option :label="$t('commons.table.all')" :value="0"></el-option>
                        <el-option
                            v-for="(group, index) in groups"
                            :key="index"
                            :label="group.name"
                            :value="group.id"
                        ></el-option>
                    </el-select>
                </div>
            </template>
            <template v-if="nginxIsExist && !openNginxConfig" #main>
                <ComplexTable
                    :pagination-config="paginationConfig"
                    :data="data"
                    @sort-change="changeSort"
                    @search="search()"
                    :class="{ mask: nginxStatus != 'Running' }"
                >
                    <el-table-column
                        :label="$t('commons.table.name')"
                        fix
                        prop="primaryDomain"
                        min-width="120px"
                        :width="mobile ? 220 : 'auto'"
                        sortable
                        show-overflow-tooltip
                    >
                        <template #default="{ row }">
                            <el-text type="primary" class="cursor-pointer" @click="openConfig(row.id)">
                                {{ row.primaryDomain }}
                            </el-text>
                        </template>
                    </el-table-column>
                    <el-table-column
                        min-width="120px"
                        :label="$t('commons.table.type')"
                        fix
                        show-overflow-tooltip
                        prop="type"
                        sortable
                    >
                        <template #default="{ row }">
                            <div v-if="row.type">
                                {{ $t('website.' + row.type) }}
                                <span v-if="row.type === 'deployment'">[{{ row.appName }}]</span>
                                <span v-if="row.type === 'runtime'">[{{ row.runtimeName }}]</span>
                            </div>
                        </template>
                    </el-table-column>
                    <el-table-column :label="$t('website.sitePath')" prop="sitePath">
                        <template #default="{ row }">
                            <el-button type="primary" link @click="toFolder(row.sitePath + '/index')">
                                <el-icon>
                                    <FolderOpened />
                                </el-icon>
                            </el-button>
                        </template>
                    </el-table-column>
                    <el-table-column :label="$t('commons.table.status')" prop="status" width="120px" sortable>
                        <template #default="{ row }">
                            <el-button
                                v-if="row.status === 'Running'"
                                link
                                type="success"
                                :icon="VideoPlay"
                                @click="opWebsite('stop', row.id)"
                            >
                                {{ $t('commons.status.running') }}
                            </el-button>
                            <el-button v-else link type="danger" :icon="VideoPause" @click="opWebsite('start', row.id)">
                                {{ $t('commons.status.stopped') }}
                            </el-button>
                        </template>
                    </el-table-column>
                    <el-table-column
                        :label="$t('website.remark')"
                        prop="remark"
                        show-overflow-tooltip
                        min-width="120px"
                    ></el-table-column>
                    <el-table-column
                        :label="$t('commons.table.protocol')"
                        prop="protocol"
                        width="90px"
                    ></el-table-column>
                    <el-table-column :label="$t('website.expireDate')">
                        <template #default="{ row, $index }">
                            <div v-show="row.showdate">
                                <el-date-picker
                                    style="width: 120px"
                                    v-model="row.expireDate"
                                    type="date"
                                    :disabled-date="checkDate"
                                    :shortcuts="shortcuts"
                                    :clearable="false"
                                    :default-value="setDate(row.expireDate)"
                                    :ref="(el) => setdateRefs(el, $index)"
                                    @change="submitDate(row)"
                                    @visible-change="(visibility:boolean) => pickerVisibility(visibility, row)"
                                    size="small"
                                ></el-date-picker>
                            </div>
                            <div v-show="!row.showdate">
                                <el-link type="primary" :underline="false" @click="openDatePicker(row, $index)">
                                    <span v-if="isEver(row.expireDate)">
                                        {{ $t('website.neverExpire') }}
                                    </span>
                                    <span v-else>
                                        {{ dateFormatSimple(row.expireDate) }}
                                    </span>
                                </el-link>
                            </div>
                        </template>
                    </el-table-column>
                    <fu-table-operations
                        :ellipsis="10"
                        width="400px"
                        :buttons="buttons"
                        :label="$t('commons.table.operate')"
                        :fixed="mobile ? false : 'right'"
                        fix
                    />
                </ComplexTable>
                <el-card width="30%" v-if="nginxStatus != 'Running' && maskShow" class="mask-prompt">
                    <span>{{ $t('commons.service.serviceNotStarted', ['OpenResty']) }}</span>
                </el-card>
            </template>
        </LayoutContent>
        <CreateWebSite ref="createRef" @close="search" />
        <DeleteWebsite ref="deleteRef" @close="search" />
        <UploadDialog ref="uploadRef" />
        <Backups ref="dialogBackupRef" />
        <DefaultServer ref="defaultRef" />
        <GroupDialog @search="listGroup" ref="groupRef" />
        <NginxConfig v-if="openNginxConfig" v-loading="loading" :containerName="containerName" :status="nginxStatus" />
    </div>
</template>

<script lang="ts" setup>
import Backups from '@/components/backup/index.vue';
import UploadDialog from '@/components/upload/index.vue';
import DefaultServer from '@/views/website/website/default/index.vue';
import { onMounted, reactive, ref, computed } from '@vue/runtime-core';
import CreateWebSite from './create/index.vue';
import DeleteWebsite from './delete/index.vue';
import GroupDialog from '@/components/group/index.vue';
import { OpWebsite, SearchWebsites, UpdateWebsite } from '@/api/modules/website';
import { Website } from '@/api/interface/website';
import AppStatus from '@/components/app-status/index.vue';
import NginxConfig from './nginx/index.vue';
import i18n from '@/lang';
import router from '@/routers';
import { App } from '@/api/interface/app';
import { ElMessageBox } from 'element-plus';
import { dateFormatSimple } from '@/utils/util';
import { MsgSuccess } from '@/utils/message';
import { useI18n } from 'vue-i18n';
import { VideoPlay, VideoPause } from '@element-plus/icons-vue';
import { GetGroupList } from '@/api/modules/group';
import { Group } from '@/api/interface/group';
import { GlobalStore } from '@/store';
const globalStore = GlobalStore();

const shortcuts = [
    {
        text: useI18n().t('website.ever'),
        value: () => {
            return new Date('1970-01-01');
        },
    },
    {
        text: i18n.global.t('website.nextYear'),
        value: () => {
            const now = new Date();
            now.setFullYear(now.getFullYear() + 1);
            return now;
        },
    },
];

const loading = ref(false);
const maskShow = ref(true);
const createRef = ref();
const deleteRef = ref();
const groupRef = ref();
const openNginxConfig = ref(false);
const nginxIsExist = ref(false);
const containerName = ref('');
const nginxStatus = ref('');
const installPath = ref('');
const uploadRef = ref();
const dialogBackupRef = ref();
const defaultRef = ref();
const data = ref();
let dateRefs: Map<number, any> = new Map();
let groups = ref<Group.GroupInfo[]>([]);

const paginationConfig = reactive({
    cacheSizeKey: 'website-page-size',
    currentPage: 1,
    pageSize: Number(localStorage.getItem('website-page-size')) || 10,
    total: 0,
});
let req = reactive({
    name: '',
    page: 1,
    pageSize: 10,
    orderBy: 'created_at',
    order: 'null',
    websiteGroupId: 0,
});
const mobile = computed(() => {
    return globalStore.isMobile();
});

const changeSort = ({ prop, order }) => {
    if (order) {
        req.orderBy = prop == 'primaryDomain' ? 'primary_domain' : prop;
        req.order = order;
    } else {
        req.orderBy = 'created_at';
        req.order = 'null';
    }
    search();
};

const search = async () => {
    req.page = paginationConfig.currentPage;
    req.pageSize = paginationConfig.pageSize;

    loading.value = true;
    await SearchWebsites(req)
        .then((res) => {
            data.value = res.data.items;
            paginationConfig.total = res.data.total;
        })
        .finally(() => {
            loading.value = false;
        });
};

const listGroup = async () => {
    const res = await GetGroupList({ type: 'website' });
    groups.value = res.data;
};

const setting = () => {
    openNginxConfig.value = true;
};

const openConfig = (id: number) => {
    router.push({ name: 'WebsiteConfig', params: { id: id, tab: 'basic' } });
};

const isEver = (time: string) => {
    const expireDate = new Date(time);
    return expireDate < new Date('1970-01-02');
};

const isBeforeNow = (time: string) => {
    return new Date() > new Date(time);
};

const setDate = (time: string) => {
    if (isEver(time)) {
        return new Date();
    } else {
        return new Date(time);
    }
};

const openDatePicker = (row: any, index: number) => {
    row.showdate = true;
    const ref = dateRefs.get(index);
    if (ref != undefined) {
        if (isBeforeNow(row.expireDate)) {
            row.oldExpireDate = row.expireDate;
            const date = new Date().toLocaleDateString();
            row.expireDate = date;
        }
        ref.handleOpen();
    }
};

const setdateRefs = (ref: any, index: number) => {
    dateRefs.set(index, ref);
};

const pickerVisibility = (visibility: boolean, row: any) => {
    if (!visibility) {
        row.showdate = false;
        if (!row.change) {
            if (row.oldExpireDate) {
                row.expireDate = row.oldExpireDate;
            }
            row.change = false;
        }
    }
};

const submitDate = (row: any) => {
    const reqDate = dateFormatSimple(row.expireDate);
    const req = {
        id: row.id,
        primaryDomain: row.primaryDomain,
        remark: row.remark,
        webSiteGroupId: row.webSiteGroupId,
        expireDate: reqDate,
        IPV6: row.IPV6,
    };

    UpdateWebsite(req).then(() => {
        row.change = true;
        MsgSuccess(i18n.global.t('commons.msg.updateSuccess'));
        search();
    });
};

const buttons = [
    {
        label: i18n.global.t('website.config'),
        click: function (row: Website.Website) {
            openConfig(row.id);
        },
    },
    {
        label: i18n.global.t('database.backupList'),
        click: (row: Website.Website) => {
            let params = {
                type: 'website',
                name: row.primaryDomain,
                detailName: row.alias,
            };
            dialogBackupRef.value!.acceptParams(params);
        },
    },
    {
        label: i18n.global.t('database.loadBackup'),
        click: (row: Website.Website) => {
            let params = {
                type: 'website',
                name: row.primaryDomain,
                detailName: row.alias,
            };
            uploadRef.value!.acceptParams(params);
        },
    },
    {
        label: i18n.global.t('commons.button.delete'),
        click: function (row: Website.Website) {
            openDelete(row);
        },
    },
];

const openDelete = (website: Website.Website) => {
    deleteRef.value.acceptParams(website);
};

const openCreate = () => {
    createRef.value.acceptParams(installPath.value);
};

const openGroup = () => {
    groupRef.value.acceptParams({ type: 'website' });
};

const openDefault = () => {
    defaultRef.value.acceptParams();
};

const checkExist = (data: App.CheckInstalled) => {
    nginxIsExist.value = data.isExist;
    containerName.value = data.containerName;
    nginxStatus.value = data.status;
    installPath.value = data.installPath;
};

const checkDate = (date: Date) => {
    const now = new Date();
    return date.getTime() < now.getTime();
};

const opWebsite = (op: string, id: number) => {
    ElMessageBox.confirm(i18n.global.t('website.' + op + 'Helper'), i18n.global.t('cronjob.changeStatus'), {
        confirmButtonText: i18n.global.t('commons.button.confirm'),
        cancelButtonText: i18n.global.t('commons.button.cancel'),
    }).then(async () => {
        await OpWebsite({ id: id, operate: op });
        MsgSuccess(i18n.global.t('commons.msg.operationSuccess'));
        search();
    });
};

const toFolder = (folder: string) => {
    router.push({ path: '/hosts/files', query: { path: folder } });
};

onMounted(() => {
    search();
    listGroup();
});
</script>
