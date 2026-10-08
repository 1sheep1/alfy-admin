<script lang="ts" setup>
import type { UploadFile, UploadFiles, UploadRawFile } from 'element-plus';

import type { BaseFacilityRecord, TeamMemberRecord } from '#/api';

import { onMounted, reactive, ref } from 'vue';

import {
  ElButton,
  ElCard,
  ElDialog,
  ElForm,
  ElFormItem,
  ElImage,
  ElInput,
  ElInputNumber,
  ElMessage,
  ElOption,
  ElPopconfirm,
  ElSelect,
  ElSwitch,
  ElTable,
  ElTableColumn,
  ElTag,
  ElUpload,
} from 'element-plus';

import {
  deleteBaseFacility,
  deleteTeamMember,
  getMediaPreviewUrl,
  listBaseFacilities,
  listMedia,
  listTeamMembers,
  saveBaseFacility,
  saveTeamMember,
  uploadMedia,
} from '#/api';
import { mediaUrl } from '#/data/cms-adapter';

// ---------- 素材选择（两个区块共用） ----------
const MAX_IMAGE_SIZE = 30 * 1024 * 1024;
const ALLOWED_IMAGE_TYPES = new Set([
  'image/gif',
  'image/jpeg',
  'image/png',
  'image/webp',
]);

interface MediaOption {
  adminUrl: string;
  id: number;
  name: string;
}

const mediaOptions = ref<MediaOption[]>([]);
const mediaLoading = ref(false);
const mediaLoaded = ref(false);
const mediaAdminUrlById = new Map<number, string>();
// 预览缓存：mediaId -> 预览地址（blob 或公开地址）
const previews = reactive<Record<number, string>>({});

async function searchMedia(keyword = '') {
  mediaLoading.value = true;
  try {
    const records = await listMedia(keyword, { page: 1, size: 100 });
    const merged = new Map(mediaOptions.value.map((item) => [item.id, item]));
    for (const item of records.filter(
      (record) => record.mediaType === 'IMAGE',
    )) {
      const option: MediaOption = {
        adminUrl: item.adminUrl,
        id: item.id,
        name: item.originalFilename,
      };
      merged.set(item.id, option);
      mediaAdminUrlById.set(item.id, item.adminUrl);
    }
    mediaOptions.value = [...merged.values()];
    mediaLoaded.value = true;
  } finally {
    mediaLoading.value = false;
  }
}

function ensureMediaOptions() {
  if (!mediaLoaded.value && !mediaLoading.value) void searchMedia();
}

function resolvePreview(mediaId?: null | number): string {
  if (!mediaId) return '';
  if (previews[mediaId]) return previews[mediaId]!;
  const adminUrl = mediaAdminUrlById.get(mediaId);
  if (!adminUrl) return mediaUrl(mediaId);
  getMediaPreviewUrl(adminUrl)
    .then((url) => {
      previews[mediaId] = url;
    })
    .catch(() => {
      previews[mediaId] = mediaUrl(mediaId);
    });
  return previews[mediaId] || mediaUrl(mediaId);
}

function validateImageFile(file: UploadRawFile) {
  if (file.size > MAX_IMAGE_SIZE) {
    ElMessage.error('单张图片不能超过 30MB');
    return false;
  }
  if (!ALLOWED_IMAGE_TYPES.has(file.type.toLowerCase())) {
    ElMessage.error('仅支持 JPG、PNG、WebP 和 GIF 图片');
    return false;
  }
  return true;
}

// ---------- 团队成员 ----------
const teamRows = ref<TeamMemberRecord[]>([]);
const teamLoading = ref(false);
const teamDialogVisible = ref(false);
const teamSaving = ref(false);
const teamUploadFileList = ref<UploadFile[]>([]);
const teamUploading = ref(false);
const teamForm = reactive({
  bio: '',
  enabled: true,
  id: 0,
  name: '',
  photoMediaId: undefined as number | undefined,
  role: '',
  sortOrder: 0,
  version: undefined as number | undefined,
});

async function loadTeam() {
  teamLoading.value = true;
  try {
    teamRows.value = await listTeamMembers();
  } finally {
    teamLoading.value = false;
  }
}

function openTeamCreate() {
  Object.assign(teamForm, {
    bio: '',
    enabled: true,
    id: 0,
    name: '',
    photoMediaId: undefined,
    role: '',
    sortOrder: (teamRows.value.at(-1)?.sortOrder ?? 0) + 10,
    version: undefined,
  });
  teamDialogVisible.value = true;
  ensureMediaOptions();
}

function openTeamEdit(value: unknown) {
  const row = value as TeamMemberRecord;
  Object.assign(teamForm, {
    ...row,
    photoMediaId: row.photoMediaId ?? undefined,
  });
  teamDialogVisible.value = true;
  ensureMediaOptions();
}

async function uploadTeamPhoto(_file: UploadFile, files: UploadFiles) {
  const latest = files.at(-1)?.raw as undefined | UploadRawFile;
  if (!latest || !validateImageFile(latest)) {
    teamUploadFileList.value = [];
    return;
  }
  teamUploading.value = true;
  try {
    const saved = await uploadMedia(
      latest,
      latest.name.replace(/\.[^.]+$/, ''),
    );
    teamForm.photoMediaId = saved.id;
    mediaAdminUrlById.set(saved.id, saved.adminUrl);
    mediaOptions.value = [
      ...mediaOptions.value.filter((item) => item.id !== saved.id),
      { adminUrl: saved.adminUrl, id: saved.id, name: saved.originalFilename },
    ];
    ElMessage.success('头像已上传并保存到素材库');
  } finally {
    teamUploading.value = false;
    teamUploadFileList.value = [];
  }
}

async function saveTeam() {
  if (!teamForm.role.trim() || !teamForm.name.trim()) {
    ElMessage.warning('请填写头衔和姓名');
    return;
  }
  teamSaving.value = true;
  try {
    await saveTeamMember(teamForm.id || null, {
      bio: teamForm.bio || null,
      enabled: teamForm.enabled,
      name: teamForm.name,
      photoMediaId: teamForm.photoMediaId ?? null,
      role: teamForm.role,
      sortOrder: teamForm.sortOrder,
      version: teamForm.version,
    });
    teamDialogVisible.value = false;
    await loadTeam();
    ElMessage.success('团队成员已保存');
  } finally {
    teamSaving.value = false;
  }
}

async function removeTeam(id: number) {
  await deleteTeamMember(id);
  await loadTeam();
  ElMessage.success('团队成员已删除');
}

// ---------- 产业布局 ----------
const facilityRows = ref<BaseFacilityRecord[]>([]);
const facilityLoading = ref(false);
const facilityDialogVisible = ref(false);
const facilitySaving = ref(false);
const facilityUploadFileList = ref<UploadFile[]>([]);
const facilityUploading = ref(false);
const facilityForm = reactive({
  address: '',
  enabled: true,
  id: 0,
  imageMediaId: undefined as number | undefined,
  name: '',
  sortOrder: 0,
  version: undefined as number | undefined,
});

async function loadFacilities() {
  facilityLoading.value = true;
  try {
    facilityRows.value = await listBaseFacilities();
  } finally {
    facilityLoading.value = false;
  }
}

function openFacilityCreate() {
  Object.assign(facilityForm, {
    address: '',
    enabled: true,
    id: 0,
    imageMediaId: undefined,
    name: '',
    sortOrder: (facilityRows.value.at(-1)?.sortOrder ?? 0) + 10,
    version: undefined,
  });
  facilityDialogVisible.value = true;
  ensureMediaOptions();
}

function openFacilityEdit(value: unknown) {
  const row = value as BaseFacilityRecord;
  Object.assign(facilityForm, {
    ...row,
    imageMediaId: row.imageMediaId ?? undefined,
  });
  facilityDialogVisible.value = true;
  ensureMediaOptions();
}

async function uploadFacilityImage(_file: UploadFile, files: UploadFiles) {
  const latest = files.at(-1)?.raw as undefined | UploadRawFile;
  if (!latest || !validateImageFile(latest)) {
    facilityUploadFileList.value = [];
    return;
  }
  facilityUploading.value = true;
  try {
    const saved = await uploadMedia(
      latest,
      latest.name.replace(/\.[^.]+$/, ''),
    );
    facilityForm.imageMediaId = saved.id;
    mediaAdminUrlById.set(saved.id, saved.adminUrl);
    mediaOptions.value = [
      ...mediaOptions.value.filter((item) => item.id !== saved.id),
      { adminUrl: saved.adminUrl, id: saved.id, name: saved.originalFilename },
    ];
    ElMessage.success('图片已上传并保存到素材库');
  } finally {
    facilityUploading.value = false;
    facilityUploadFileList.value = [];
  }
}

async function saveFacility() {
  if (!facilityForm.name.trim()) {
    ElMessage.warning('请填写基地名称');
    return;
  }
  facilitySaving.value = true;
  try {
    await saveBaseFacility(facilityForm.id || null, {
      address: facilityForm.address || null,
      enabled: facilityForm.enabled,
      imageMediaId: facilityForm.imageMediaId ?? null,
      name: facilityForm.name,
      sortOrder: facilityForm.sortOrder,
      version: facilityForm.version,
    });
    facilityDialogVisible.value = false;
    await loadFacilities();
    ElMessage.success('产业布局已保存');
  } finally {
    facilitySaving.value = false;
  }
}

async function removeFacility(id: number) {
  await deleteBaseFacility(id);
  await loadFacilities();
  ElMessage.success('产业布局已删除');
}

onMounted(() => {
  void loadTeam();
  void loadFacilities();
  ensureMediaOptions();
});
</script>

<template>
  <div class="about-page">
    <section class="page-header">
      <div>
        <p>ABOUT PAGE</p>
        <h1>关于我们页面</h1>
        <span>维护官网关于我们页面的核心研发团队与产业布局内容</span>
      </div>
    </section>

    <ElCard class="table-card" shadow="never" v-loading="teamLoading">
      <div class="card-toolbar">
        <div class="card-title">
          <h2>核心研发团队</h2>
          <span>按排序值展示，官网轮播顺序与此一致</span>
        </div>
        <ElButton type="primary" @click="openTeamCreate">新增成员</ElButton>
      </div>
      <ElTable :data="teamRows" row-key="id">
        <ElTableColumn label="头像" width="100">
          <template #default="{ row }">
            <ElImage
              v-if="row.photoMediaId"
              :preview-src-list="[resolvePreview(row.photoMediaId)]"
              :src="resolvePreview(row.photoMediaId)"
              class="team-thumb"
              fit="cover"
            />
            <div v-else class="thumb-placeholder">未设置</div>
          </template>
        </ElTableColumn>
        <ElTableColumn label="头衔" prop="role" width="150" />
        <ElTableColumn label="姓名" prop="name" width="140" />
        <ElTableColumn
          label="个人简介"
          min-width="260"
          prop="bio"
          show-overflow-tooltip
        />
        <ElTableColumn label="状态" width="90">
          <template #default="{ row }">
            <ElTag :type="row.enabled ? 'success' : 'info'">
              {{ row.enabled ? '显示' : '隐藏' }}
            </ElTag>
          </template>
        </ElTableColumn>
        <ElTableColumn label="排序" prop="sortOrder" width="80" />
        <ElTableColumn fixed="right" label="操作" width="150">
          <template #default="{ row }">
            <ElButton link type="primary" @click="openTeamEdit(row)">
              编辑
            </ElButton>
            <ElPopconfirm
              title="确定删除该团队成员？"
              @confirm="removeTeam(row.id)"
            >
              <template #reference>
                <ElButton link type="danger">删除</ElButton>
              </template>
            </ElPopconfirm>
          </template>
        </ElTableColumn>
      </ElTable>
    </ElCard>

    <ElCard class="table-card" shadow="never" v-loading="facilityLoading">
      <div class="card-toolbar">
        <div class="card-title">
          <h2>发展引擎 · 产业布局</h2>
          <span>官网“构建研产销一体化产业布局”板块的基地卡片</span>
        </div>
        <ElButton type="primary" @click="openFacilityCreate">新增基地</ElButton>
      </div>
      <ElTable :data="facilityRows" row-key="id">
        <ElTableColumn label="图片" width="160">
          <template #default="{ row }">
            <ElImage
              v-if="row.imageMediaId"
              :preview-src-list="[resolvePreview(row.imageMediaId)]"
              :src="resolvePreview(row.imageMediaId)"
              class="facility-thumb"
              fit="cover"
            />
            <div v-else class="thumb-placeholder">未设置</div>
          </template>
        </ElTableColumn>
        <ElTableColumn label="基地名称" min-width="220" prop="name" />
        <ElTableColumn
          label="地址"
          min-width="260"
          prop="address"
          show-overflow-tooltip
        />
        <ElTableColumn label="状态" width="90">
          <template #default="{ row }">
            <ElTag :type="row.enabled ? 'success' : 'info'">
              {{ row.enabled ? '显示' : '隐藏' }}
            </ElTag>
          </template>
        </ElTableColumn>
        <ElTableColumn label="排序" prop="sortOrder" width="80" />
        <ElTableColumn fixed="right" label="操作" width="150">
          <template #default="{ row }">
            <ElButton link type="primary" @click="openFacilityEdit(row)">
              编辑
            </ElButton>
            <ElPopconfirm
              title="确定删除该基地？"
              @confirm="removeFacility(row.id)"
            >
              <template #reference>
                <ElButton link type="danger">删除</ElButton>
              </template>
            </ElPopconfirm>
          </template>
        </ElTableColumn>
      </ElTable>
    </ElCard>

    <ElDialog
      v-model="teamDialogVisible"
      :close-on-click-modal="false"
      :title="teamForm.id ? '编辑团队成员' : '新增团队成员'"
      width="640px"
    >
      <ElForm :model="teamForm" label-position="top">
        <ElRow :gutter="16">
          <ElCol :md="8" :xs="24">
            <ElFormItem label="头衔" required>
              <ElInput
                v-model="teamForm.role"
                maxlength="100"
                placeholder="例如：技术带头人"
              />
            </ElFormItem>
          </ElCol>
          <ElCol :md="8" :xs="24">
            <ElFormItem label="姓名" required>
              <ElInput v-model="teamForm.name" maxlength="100" />
            </ElFormItem>
          </ElCol>
          <ElCol :md="8" :xs="24">
            <ElFormItem label="排序">
              <ElInputNumber
                v-model="teamForm.sortOrder"
                :min="0"
                style="width: 100%"
              />
            </ElFormItem>
          </ElCol>
        </ElRow>
        <ElFormItem label="个人简介">
          <ElInput
            v-model="teamForm.bio"
            :rows="3"
            maxlength="1000"
            show-word-limit
            type="textarea"
          />
        </ElFormItem>
        <ElFormItem label="头像">
          <div class="image-field">
            <ElSelect
              v-model="teamForm.photoMediaId"
              :loading="mediaLoading"
              :remote-method="searchMedia"
              allow-create
              clearable
              filterable
              placeholder="从素材库选择"
              remote
              style="width: 100%"
              @visible-change="
                (visible: boolean) => visible && ensureMediaOptions()
              "
            >
              <ElOption
                v-for="item in mediaOptions"
                :key="item.id"
                :label="item.name"
                :value="item.id"
              />
            </ElSelect>
            <ElUpload
              v-model:file-list="teamUploadFileList"
              :auto-upload="false"
              :on-change="uploadTeamPhoto"
              :show-file-list="false"
              accept=".jpg,.jpeg,.png,.webp,.gif"
            >
              <ElButton :loading="teamUploading">从本地上传</ElButton>
            </ElUpload>
          </div>
          <ElImage
            v-if="teamForm.photoMediaId"
            :preview-src-list="[resolvePreview(teamForm.photoMediaId)]"
            :src="resolvePreview(teamForm.photoMediaId)"
            class="dialog-preview"
            fit="cover"
          />
        </ElFormItem>
        <ElFormItem label="官网显示">
          <ElSwitch v-model="teamForm.enabled" />
        </ElFormItem>
      </ElForm>
      <template #footer>
        <ElButton @click="teamDialogVisible = false">取消</ElButton>
        <ElButton :loading="teamSaving" type="primary" @click="saveTeam">
          保存
        </ElButton>
      </template>
    </ElDialog>

    <ElDialog
      v-model="facilityDialogVisible"
      :close-on-click-modal="false"
      :title="facilityForm.id ? '编辑基地' : '新增基地'"
      width="640px"
    >
      <ElForm :model="facilityForm" label-position="top">
        <ElRow :gutter="16">
          <ElCol :md="16" :xs="24">
            <ElFormItem label="基地名称" required>
              <ElInput
                v-model="facilityForm.name"
                maxlength="100"
                placeholder="例如：湖南省浏阳市研发基地"
              />
            </ElFormItem>
          </ElCol>
          <ElCol :md="8" :xs="24">
            <ElFormItem label="排序">
              <ElInputNumber
                v-model="facilityForm.sortOrder"
                :min="0"
                style="width: 100%"
              />
            </ElFormItem>
          </ElCol>
        </ElRow>
        <ElFormItem label="地址">
          <ElInput
            v-model="facilityForm.address"
            :rows="2"
            maxlength="200"
            show-word-limit
            type="textarea"
          />
        </ElFormItem>
        <ElFormItem label="展示图片">
          <div class="image-field">
            <ElSelect
              v-model="facilityForm.imageMediaId"
              :loading="mediaLoading"
              :remote-method="searchMedia"
              allow-create
              clearable
              filterable
              placeholder="从素材库选择"
              remote
              style="width: 100%"
              @visible-change="
                (visible: boolean) => visible && ensureMediaOptions()
              "
            >
              <ElOption
                v-for="item in mediaOptions"
                :key="item.id"
                :label="item.name"
                :value="item.id"
              />
            </ElSelect>
            <ElUpload
              v-model:file-list="facilityUploadFileList"
              :auto-upload="false"
              :on-change="uploadFacilityImage"
              :show-file-list="false"
              accept=".jpg,.jpeg,.png,.webp,.gif"
            >
              <ElButton :loading="facilityUploading">从本地上传</ElButton>
            </ElUpload>
          </div>
          <ElImage
            v-if="facilityForm.imageMediaId"
            :preview-src-list="[resolvePreview(facilityForm.imageMediaId)]"
            :src="resolvePreview(facilityForm.imageMediaId)"
            class="dialog-preview"
            fit="cover"
          />
        </ElFormItem>
        <ElFormItem label="官网显示">
          <ElSwitch v-model="facilityForm.enabled" />
        </ElFormItem>
      </ElForm>
      <template #footer>
        <ElButton @click="facilityDialogVisible = false">取消</ElButton>
        <ElButton
          :loading="facilitySaving"
          type="primary"
          @click="saveFacility"
        >
          保存
        </ElButton>
      </template>
    </ElDialog>
  </div>
</template>

<style scoped>
.about-page {
  min-height: 100%;
  padding: 24px;
  background: #f5f7f8;
}

.page-header {
  padding: 28px 30px;
  color: #fff;
  background: linear-gradient(125deg, #102a35, #125b61);
  border-radius: 18px;
}

.page-header p {
  margin: 0;
  font-size: 11px;
  font-weight: 700;
  color: #78d2c8;
  letter-spacing: 0.18em;
}

.page-header h1 {
  margin: 5px 0;
  font-size: 28px;
}

.page-header span {
  color: rgb(255 255 255 / 68%);
}

.table-card {
  margin-top: 16px;
  border-radius: 16px;
}

.card-toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 16px;
}

.card-title h2 {
  margin: 0;
  font-size: 18px;
  color: #17363d;
}

.card-title span {
  font-size: 12px;
  color: #859398;
}

.team-thumb {
  width: 64px;
  height: 80px;
  border-radius: 8px;
}

.facility-thumb {
  width: 128px;
  height: 80px;
  border-radius: 8px;
}

.thumb-placeholder {
  display: grid;
  place-items: center;
  width: 64px;
  height: 80px;
  font-size: 12px;
  color: #9aa8ac;
  background: #eef2f3;
  border-radius: 8px;
}

.image-field {
  display: flex;
  gap: 10px;
  width: 100%;
}

.dialog-preview {
  width: 160px;
  height: 120px;
  margin-top: 10px;
  border-radius: 10px;
}

@media (max-width: 760px) {
  .about-page {
    padding: 14px;
  }

  .page-header {
    padding: 22px;
  }

  .image-field {
    flex-direction: column;
  }
}
</style>
