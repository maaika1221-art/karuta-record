<script setup lang="ts">
const gradeItems = ref(['A級六段','A級五段','A級四段','B級参段','C級弍段','D級初段','E級','F級','G級'])
const gradeValue = ref('A級四段')
const nameItmes = ref(['佐藤花','鈴木優','小林蓮'])
const nameValue = ref('')
const winorlossItems = ref(['勝ち','負け'])
const winorlossValue =('勝ち')
</script>

<template>
<h1>記録</h1>
<p>対戦日</p>
<UInput type="date" />
<p>対戦相手の級・段位</p>
<USelectMenu v-model="gradeValue":items="gradeItems" />
<p>対戦相手の名前</p>
<UInput placeholder="名前を入力してください。" v-model="nameValue":items="nameItmes" />
<p>結果</p>
<URadioGroup v-model="winorlossValue":items="winorlossItems" />

</template>