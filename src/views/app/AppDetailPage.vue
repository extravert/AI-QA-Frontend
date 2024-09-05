<template>
  <div id="appDetailPage">
    <a-card>
      <a-row style="margin-bottom: 16px">
        <a-col flex="auto" class="content-wrapper">
          <h2>{{ data.appName }}</h2>
          <p>{{ data.appDesc }}</p>
          <p>应用类型：{{ APP_TYPE_MAP[data.appType] }}</p>
          <p>评分策略：{{ APP_SCORING_STRATEGY_MAP[data.scoringStrategy] }}</p>
          <p>
            <a-space>
              作者：
              <div :style="{ display: 'flex', alignItems: 'center' }">
                <a-avatar
                  :size="24"
                  :image-url="data.user?.userAvatar"
                  :style="{ marginRight: '8px' }"
                />
                <a-typography-text
                  >{{ data.user?.userName ?? "无名" }}
                </a-typography-text>
              </div>
            </a-space>
          </p>
          <p>
            创建时间：{{ dayjs(data.createTime).format("YYYY-MM-DD HH:mm:ss") }}
          </p>
          <a-space size="medium">
            <a-button type="primary" :href="`/answer/do/${id}`"
              >开始答题
            </a-button>
            <a-button @click="doShare">分享应用</a-button>
            <a-button v-if="isMy" :href="`/add/question/${id}`"
              >设置题目
            </a-button>
            <a-button v-if="isMy" :href="`/add/scoring_result/${id}`"
              >设置评分
            </a-button>
            <a-button v-if="isMy" :href="`/add/app/${id}`">修改应用</a-button>
          </a-space>
        </a-col>
        <a-col flex="320px">
          <a-image width="100%" :src="data.appIcon" />
        </a-col>
      </a-row>
    </a-card>
    <ShareModal :link="shareLink" title="应用分享" ref="shareModalRef" />
  <!-- 添加评论区 -->
  <div class="comment-section">
      <h3>评论区 ({{ comments.length }}条评论)</h3>
      <div v-if="comments.length === 0">暂无评论</div>
      <div v-else>
        <div v-for="(comment, index) in comments" :key="index" class="comment">
          <div class="comment-header">
            <img :src="comment.avatar" alt="用户头像" class="avatar">
            <span class="author">{{ comment.author }}</span>
            <span class="datetime">{{ comment.datetime }}</span>
          </div>
          <div class="comment-content">
            {{ comment.content }}
          </div>
          <div class="comment-actions">
            <a-space>
              <a-button 
                size="small" 
                @click="likeComment(index)"
                :class="{ 'liked': comment.liked }"
              >
                点赞 {{ comment.likes }}
              </a-button>
              <a-button 
                size="small" 
                @click="dislikeComment(index)"
                :class="{ 'disliked': comment.disliked }"
              >
                点踩 {{ comment.dislikes }}
              </a-button>
              <a-button size="small" @click="replyToComment(index)">
                回复
              </a-button>
            </a-space>
          </div>
        </div>
      </div>
    </div>
    <!-- 添加评论表单 -->
    <div class="comment-form">
      <a-input
        v-model="newComment"
        placeholder="添加新回复..."
        :rows="4"
        type="textarea"
      />
      <a-button type="primary" @click="handleSubmit" style="margin-top: 10px;">
        发表评论
      </a-button>
    </div>

    <!-- 停靠的回复框 -->
    <div id="reply-box" :class="{ 'docked': isReplyBoxDocked,'comment-box-width': true }">
      <div class="reply-box-header">
        <div @click="submitReply" >回复 {{ replyingTo }}</div>
        <div>
          <a-button size="small" @click="undockReplyBox">取消回复框停靠</a-button>
          <a-button size="small" @click="goToTop">回到顶部</a-button>
        </div>
      </div>
      <a-textarea
        v-model="replyContent"
        :placeholder="`回复 ${replyingTo}`"
        :rows="4"
      />
      <a-button type="primary" @click="submitReply" style="margin-top: 10px;">
        提交回复
      </a-button>
    </div>
  </div>

</template>

<script setup lang="ts">
import { Message } from '@arco-design/web-vue';

import { onMounted, computed, defineProps, ref, watchEffect, withDefaults } from "vue";
import API from "@/api";
import { getAppVoByIdUsingGet } from "@/api/appController";
import message from "@arco-design/web-vue/es/message";
import { useRouter } from "vue-router";
import { dayjs } from "@arco-design/web-vue/es/_utils/date";
import { useLoginUserStore } from "@/store/userStore";
import { APP_SCORING_STRATEGY_MAP, APP_TYPE_MAP } from "@/constant/app";
import ShareModal from "@/components/ShareModal.vue";

interface Props {
  id: string;
}

const props = withDefaults(defineProps<Props>(), {
  id: () => {
    return "";
  },
});

const router = useRouter();

const data = ref<API.AppVO>({});

// 获取登录用户
const loginUserStore = useLoginUserStore();
let loginUserId = loginUserStore.loginUser?.id;
console.log(loginUserId, data.value.userId, 12354325);

// 是否为本人创建
const isMy = computed(() => {
  return loginUserId && loginUserId === data.value.userId;
});

/**
 * 加载数据
 */
const loadData = async () => {
  if (!props.id) {
    return;
  }
  const res = await getAppVoByIdUsingGet({
    id: props.id as any,
  });
  if (res.data.code === 0) {
    data.value = res.data.data as any;
  } else {
    message.error("获取数据失败，" + res.data.message);
  }
};

/**
 * 监听 searchParams 变量，改变时触发数据的重新加载
 */
watchEffect(() => {
  loadData();
});

// 分享弹窗的引用
const shareModalRef = ref();

// 分享链接
const shareLink = `${window.location.protocol}//${window.location.host}/app/detail/${props.id}`;

// 分享
const doShare = (e: Event) => {
  if (shareModalRef.value) {
    shareModalRef.value.openModal();
  }
  // 阻止冒泡，防止跳转到详情页
  e.stopPropagation();
};

// 添加评论数据
const comments = ref([
  {
    author: '用户1',
    avatar: '	https://cdn.v2ex.com/gravatar/9464475c753203819f8c040ae347fbf7?s=48&d=retro',
    content: '这是一个很棒的应用!',
    datetime: dayjs().subtract(2, 'days').format('YYYY-MM-DD HH:mm:ss'),
    likes: 5,
    dislikes: 1,
    liked: false,
    disliked: false,
  },
  {
    author: '用户2',
    avatar: '	https://cdn.v2ex.com/gravatar/9464475c753203819f8c040ae347fbf7?s=48&d=retro',
    content: '我觉得还可以改进一下用户界面。',
    datetime: dayjs().subtract(1, 'days').format('YYYY-MM-DD HH:mm:ss'),
    likes: 3,
    dislikes: 0,
    liked: false,
    disliked: false,
  },
  {
    author: '用户3',
    avatar: '	https://cdn.v2ex.com/gravatar/9464475c753203819f8c040ae347fbf7?s=48&d=retro',
    content: '期待下一个版本的更新!',
    datetime: dayjs().format('YYYY-MM-DD HH:mm:ss'),
    likes: 8,
    dislikes: 2,
    liked: false,
    disliked: false,
  },
  {
    author: '用户3',
    avatar: '	https://cdn.v2ex.com/gravatar/9464475c753203819f8c040ae347fbf7?s=48&d=retro',
    content: '期待下一个版本的更新!',
    datetime: dayjs().format('YYYY-MM-DD HH:mm:ss'),
    likes: 8,
    dislikes: 2,
    liked: false,
    disliked: false,
  },
]);

// 新评论输入
const newComment = ref('');

// 提交评论的方法
const handleSubmit = () => {
  if (!loginUserId) {
      Message.warning("请先登录");
      window.location.href = `/user/login?redirect=${window.location.href}`;
      return;
    }
  if (newComment.value.trim()) {
    comments.value.push({
      author: '当前用户',
      avatar: 'https://joeschmoe.io/api/v1/random',
      content: newComment.value,
      datetime: dayjs().format('YYYY-MM-DD HH:mm:ss'),
      likes: 0,
      dislikes: 0,
      liked: false,
      disliked: false,
    });
    newComment.value = '';
    Message.success('评论提交成功');
  } else {
    Message.warning('评论内容不能为空');
  }
};


const likeComment = (index) => {
  const comment = comments.value[index];
  if (!comment.liked) {
    comment.likes++;
    comment.liked = true;
    if (comment.disliked) {
      comment.dislikes--;
      comment.disliked = false;
    }
    Message.success('点赞成功');
  } else {
    comment.likes--;
    comment.liked = false;
    Message.info('取消点赞');
  }
};

const dislikeComment = (index) => {
  const comment = comments.value[index];
  if (!comment.disliked) {
    comment.dislikes++;
    comment.disliked = true;
    if (comment.liked) {
      comment.likes--;
      comment.liked = false;
    }
    Message.success('点踩成功');
  } else {
    comment.dislikes--;
    comment.disliked = false;
    Message.info('取消点踩');
  }
};

const isReplyBoxDocked = ref(false);
const replyingTo = ref('');
const replyContent = ref('');

const replyToComment = (index) => {
  const comment = comments.value[index];
  replyingTo.value = comment.author;
  isReplyBoxDocked.value = true;
  replyContent.value = '';
};

const undockReplyBox = () => {
  isReplyBoxDocked.value = false;
  replyingTo.value = '';
  replyContent.value = '';
};

const goToTop = () => {
  window.scrollTo({ top: 0, behavior: 'smooth' });
};

const submitReply = () => {
  if (!loginUserId) {
      Message.warning("请先登录");
      window.location.href = `/user/login?redirect=${window.location.href}`;
      return;
    }
  if (replyContent.value.trim()) {
    comments.value.push({
      author: '当前用户',
      avatar: 'https://joeschmoe.io/api/v1/random',
      content: `回复 @${replyingTo.value}: ${replyContent.value}`,
      datetime: dayjs().format('YYYY-MM-DD HH:mm:ss'),
      likes: 0,
      dislikes: 0,
      liked: false,
      disliked: false,
    });
    Message.success('回复提交成功');
    undockReplyBox();
  } else {
    Message.warning('回复内容不能为空');
  }
};

onMounted(() => {
  console.log('组件挂载完成，评论数据:', comments.value);
});
</script>

<style scoped>
#appDetailPage {
}

#appDetailPage .content-wrapper > * {
  margin-bottom: 24px;
}
.comment-section {
  margin-top: 20px;
}

.comment {
  border-bottom: 1px solid #eee;
  padding: 15px 0;
}

.comment-header {
  display: flex;
  align-items: center;
  margin-bottom: 10px;
}

.avatar {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  margin-right: 10px;
}

.author {
  font-weight: bold;
  margin-right: 10px;
}

.datetime {
  color: #999;
  font-size: 0.9em;
}

.comment-content {
  margin-bottom: 10px;
}

.comment-actions {
  display: flex;
  justify-content: flex-end;
}

.comment-form {
  margin-top: 20px;
}


#reply-box {
  position: fixed;
  bottom: -200px; /* 初始状态隐藏在屏幕底部下方 */
  left: 0;
  right: 0;
  background-color: white;
  padding: 20px;
  box-shadow: 0 -2px 10px rgba(0, 0, 0, 0.1);
  transition: bottom 0.3s ease-in-out;
}

#reply-box.docked {
  bottom: 0; /* 停靠状态 */
}

.reply-box-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
}

.comment-box-width {
  width: 100%; /* 或者与评论框相同的具体宽度 */
  max-width: 1130px; /* 根据需要调整 */
  margin: 0 auto; /* 居中显示 */
}
.liked {
  color: #f53f3f !important;
}

.disliked {
  color: #165DFF !important;
}
</style>
