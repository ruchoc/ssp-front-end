<template>
  <a-list item-layout="horizontal" :data-source="shareData.shareList">
    <template #renderItem="{ item }">
      <a-list-item>
        <a-comment style="width: 100%">
          <template #author>{{ item.username }}</template>
          <template #avatar>
            <a-avatar
              @click="
                $router.push({
                  path: '/person',
                  query: { id: item.userId },
                })
              "
              :src="getAvatarUrl(item.userAvatarUrl)"
            />
          </template>
          <template #content>
            <p class="content">
              {{ item.content }}
            </p>
          </template>
          <template #datetime>
            <a-tooltip :title="dayjs(item.time).format('YYYY-MM-DD HH:mm:ss')">
              <span>{{ dayjs(item.time).fromNow() }}</span>
            </a-tooltip>
          </template>
          <template #actions>
            <span
              @click="
                $router.push({
                  path: '/share',
                  query: { id: item.id },
                })
              "
              >详情</span
            >
            <span @click="openLikeDetail(item.id)" style="padding-left: 16px"
              >点赞：{{ item.likeNum }}</span
            >
          </template>
          <div class="but-box">
            <a-button type="primary" @click="openModal(item)">编辑</a-button>
            <a-button type="primary" danger @click="onDelete(item.id)"
              >删除</a-button
            >
          </div>
        </a-comment>
      </a-list-item>
    </template>
  </a-list>
  <div class="clearfix">
    <div class="pagination">
      <a-pagination
        v-model:current="current"
        v-model:total="shareData.shareTotal"
        v-model:pageSize="pageSize"
        show-less-items
        hideOnSinglePage
        @change="pageChange"
      />
    </div>
  </div>
  <a-modal
    v-model:visible="modalVisible"
    title="编辑分享"
    okText="提交"
    cancelText="取消"
    @ok="onEdit"
  >
    <a-form
      :model="formState"
      :label-col="{ span: 4 }"
      :wrapper-col="{ span: 16 }"
      autocomplete="off"
      ref="form"
    >
      <a-form-item
        label="内容"
        name="content"
        :rules="[
          { required: true, message: '请输入分享内容', trigger: 'blur' },
        ]"
      >
        <a-textarea style="height: 140px" v-model:value="formState.content" />
      </a-form-item>
      <a-form-item label="是否公开">
        <a-switch v-model:checked="checked"></a-switch>
      </a-form-item>
      <a-form-item label="上传图片">
        <Image
          v-if="modalVisible"
          ref="image"
          :pictureList="formState.pictureList"
        ></Image>
      </a-form-item>
    </a-form>
  </a-modal>
  <a-drawer
    v-model:visible="drawerVisible"
    title="点赞的用户"
    placement="right"
    :closable="false"
  >
    <a-list item-layout="horizontal" :data-source="likeUserList">
      <template #renderItem="{ item }">
        <a-list-item>
          <a-list-item-meta>
            <template #title>
              <span class="username">{{ item.username }}</span>
            </template>
            <template #avatar>
              <router-link
                target="_blank"
                :to="{ name: 'person', query: { id: item.userId } }"
                ><a-avatar :src="getAvatarUrl(item.avatarUrl)"
              /></router-link>
            </template>
          </a-list-item-meta>
        </a-list-item>
      </template>
    </a-list>
  </a-drawer>
</template>
<script setup>
import { getLikeUserList } from "@/hooks/action";
import Image from "@/components/Update/Image.vue";
import dayjs from "@/global/dayjs";
import { onMounted, ref, reactive, computed } from "vue";
import {
  getMyShareList,
  getMyShareTotal,
  shareData,
  deleteShare,
  updateShare,
  setShareState,
} from "@/hooks/share";
import { message } from "ant-design-vue";
import { pageSize } from "@/global/config";
import { getAvatarUrl, userData } from "@/hooks/user";

const likeUserList = ref([]);
const drawerVisible = ref(false);
const checked = computed({
  get() {
    return formState.state !== "private";
  },
  set(v) {
    formState.state = v ? "public" : "private";
  },
});
const formState = reactive({
  id: "",
  content: "",
  state: "public",
  pictureList: [],
});
const current = ref(1);
const modalVisible = ref(false);
const form = ref(null);
const image = ref(null);

const openLikeDetail = async (shareId) => {
  try {
    likeUserList.value = await getLikeUserList(shareId);
    console.log(likeUserList.value);
    drawerVisible.value = true;
  } catch (err) {
    console.error(err);
    message.error("获取点赞信息失败");
  }
};
const openModal = (share) => {
  formState.id = share.id;
  formState.content = share.content;
  formState.state = share.state;
  formState.pictureList = share.pictureList;
  modalVisible.value = true;
};
const onEdit = async () => {
  if (!userData.isLogin) {
    message.info("请先登录");
    return;
  }
  try {
    await form.value.validateFields();
  } catch (err) {
    return;
  }
  try {
    await updateShare(formState.id, formState.content);
    await setShareState(formState.id, formState.state);
    await image.value.handleUpload(formState.id);
    getPage();
  } catch (err) {
    console.error(err);
    message.error("编辑分享失败");
  } finally {
    modalVisible.value = false;
  }
};
const onDelete = async (shareId) => {
  if (!userData.isLogin) {
    message.info("请先登录");
    return;
  }
  try {
    await deleteShare(shareId);
    getPage();
  } catch (err) {
    console.error(err);
    message.error("删除分享失败");
  }
};
const getPage = async () => {
  if (!userData.isLogin) {
    message.info("请先登录");
    return;
  }
  try {
    await getMyShareList(current.value, pageSize);
    await getMyShareTotal();
  } catch (err) {
    console.error(err);
    message.error("获取分享列表失败");
  }
};
const pageChange = () => {
  getPage();
};
onMounted(() => {
  getPage();
});
</script>

<style lang="less" scoped>
.username {
  position: relative;
  top: 5px;
}
.content {
  margin-right: 160px;
}
.pagination {
  float: right;
  padding: 16px 0 125px;
}
.but-box {
  position: absolute;
  right: 0;
  top: 18px;
  display: flex;
  gap: 10px;
}
</style>
