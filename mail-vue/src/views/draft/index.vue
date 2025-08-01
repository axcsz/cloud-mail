<template>
  <emailScroll ref="scroll"
               :allow-star="false"
               :cancel-success="cancelStar"
               :getEmailList="getEmailList"
               :emailDelete="emailDelete"
               :star-add="starAdd"
               :star-cancel="starCancel"
               @jump="jumpContent"
               actionLeft="6px"
               :show-account-icon="false"
               :showStar="false"
               @delete-draft="deleteDraft"
               :type="'draft'"
  >
    <template #name="props">
      <span class="send-email" >{{props.email.receiveEmail.join(',') || '('+$t('noRecipient')+')'}}</span>
    </template>
    <template #subject="props" >
      {{props.email.subject || '('+$t('noSubject')+')'}}
    </template>
  </emailScroll>
</template>

<script setup>
import emailScroll from "@/components/email-scroll/index.vue"
import {emailDelete} from "@/request/email.js";
import {starAdd, starCancel} from "@/request/star.js";
import {useEmailStore} from "@/store/email.js";
import {defineOptions, onMounted, ref, watch, toRaw} from "vue";
import {useUiStore} from "@/store/ui.js";
import {userDraftStore} from "@/store/draft.js";
import db from "@/db/db.js"

defineOptions({
  name: 'draft'
})

const draftStore = userDraftStore();
const uiStore = useUiStore();
const scroll = ref({})
const emailStore = useEmailStore();

watch(() => draftStore.setDraft, async () => {

  const draft = toRaw(draftStore.setDraft)
  const draftId = draft.draftId
  const attachments = toRaw(draftStore.setDraft.attachments)

  delete draft.draftId
  delete draft.attachments

  if (!draft.content && !draft.subject && !(draft.receiveEmail.length > 0)) {
    await db.value.draft.delete(draftId);
    await db.value.att.delete(draftId);
    scroll.value.refreshList();
    return;
  }

  await db.value.draft.update(draftId, draft);
  await db.value.att.update(draftId, {attachments: attachments});
  scroll.value.refreshList();
},{
  deep: true
})

watch(() => draftStore.refreshList,() => {
  scroll.value.refreshList();
})

function getEmailList() {
  return new Promise((resolve, reject) => {
    db.value.draft.orderBy('createTime').reverse().toArray().then(list => {
      resolve({list})
    })
  })
}

async function deleteDraft(draftIds) {
  await db.value.draft.bulkDelete(draftIds);
  scroll.value.refreshList();
}

async function jumpContent(email) {
  const att = await db.value.att.get(email.draftId)
  email.attachments = att.attachments
  uiStore.writerRef.openDraft(email);
}

function cancelStar(email) {
  emailStore.cancelStarEmailId = email.emailId
  scroll.value.deleteEmail([email.emailId])
}

onMounted(() => {
  emailStore.starScroll = scroll
})

</script>
<style>
.send-email {
  font-weight: normal;
}
</style>