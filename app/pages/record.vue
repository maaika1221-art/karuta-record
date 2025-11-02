<script setup lang="ts">
const recordItems = ref(['対戦','指導','一人練習'])
const recordvalue = ref('対戦')
const matchnumberValue = ref('')
const gradeItems = ref(['A級六段','A級五段','A級四段','B級参段','C級弍段','D級初段','E級','F級','G級'])
const gradeValue = ref('A級四段')
const nameItmes = ref(['佐藤花','鈴木優','小林蓮'])
const nameValue = ref('')
const winorlossItems = ref(['勝ち','負け'])
const winorlossValue = ref('勝ち')
const numberValue = ref('')

const createInitialCardPlacements = () => {
    // 6段 x 16列 = 96マス
    // status: 'E' (Empty), 'S' (Self Acquired), 'O' (Opponent Acquired), 'F' (Fault/Otsuki)
    return Array(6 * 16).fill(null).map((_, index) => ({
        id: index,
        row: Math.floor(index / 16),
        col: index % 16,
        status: 'E',
        cardName: '',     // 札名
        acquiredBy: '',   // 取得者 ('Self' | 'Opponent')
        isFault: false,   // お手つきフラグ
    }));
};

const cardPlacements = ref(createInitialCardPlacements());

const showInputModal = ref(false);
const editingCardId = ref<number|null>(null); // 現在編集中の空マスID (複数枚入力時は最初のマスID)
const newCardEntry = ref({
    cardNames: '', // カンマ区切りで複数枚の札名を入力
    acquiredBy: 'Self', // 初期値は自取
    isFault: false,
});
const savedMatches = ref< {date: string;
    placedCards: {
        id: number;
        row: number;
        col: number;
        status: string;
        cardName: string;
        acquiredBy: string;
        isFault: boolean;
    }[];
    selfScore: number;
    opponentScore: number;
    faultCount: number;}[]>([]); // 保存された試合記録を保持する配列

/**
 * 札が配置されていないマスがクリックされたときの処理
 */
const handleCardClick = (cardId:number) => {
    const clickedCard = cardPlacements.value.find(card => card.id === cardId);
    if(!clickedCard){
        return
    }

    if (clickedCard.status === 'E') {
        // 札が空の場合: 入力モーダルを開く
        editingCardId.value = cardId;
        newCardEntry.value.cardNames = ''; // リセット
        newCardEntry.value.acquiredBy = 'Self';
        newCardEntry.value.isFault = false;
        showInputModal.value = true;
    } else {
        // 札が配置されている場合: 状態を入力
        toggleCardStatus(cardId);
    }
};

/**
 * 札が既に配置されている場合に状態を入力するロジック
 */
const toggleCardStatus = (cardId:number) => {
    cardPlacements.value = cardPlacements.value.map(card => {
        if (card.id === cardId) {
            // 入力順: 自取(S) -> 敵取(O) -> お手つき(F) -> 空(E)
            if (card.status === 'S') {
                return { ...card, status: 'O', acquiredBy: 'Opponent', isFault: false };
            } else if (card.status === 'O') {
                return { ...card, status: 'F', acquiredBy: 'Self', isFault: true }; // お手つきの場合、取得は Self と見なす
            } else if (card.status === 'F') {
                return { ...card, status: 'E', acquiredBy: '', cardName: '', kimariji: '', isFault: false }; // 空に戻す
            }
        }
        return card;
    });
};

/**
 * モーダルからの入力を保存し、グリッドに反映する
 * 複数枚入力に対応するため、空いているマスに順次配置
 */
const saveCardEntry = () => {
    const cardNamesInput = newCardEntry.value.cardNames.split(',').map(name => name.trim()).filter(name => name !== '');

    if (cardNamesInput.length === 0) {
        alert('札名を入力してください。');
        return;
    }

    const { acquiredBy, isFault } = newCardEntry.value;
    const status = isFault ? 'F' : (acquiredBy === 'Self' ? 'S' : 'O');

    let currentCardIndex = cardPlacements.value.findIndex(card => card.id === editingCardId.value);
    let placementsToUpdate:typeof cardPlacements["value"] = [];

    // 入力された札名ごとに、空いているマスに順次配置する
    cardNamesInput.forEach(cardName => {
        // 現在の編集位置、または次の空いているマスを探す
        let targetIndex = currentCardIndex;
        const currentCardPlacement =cardPlacements.value[targetIndex]
        if (!currentCardPlacement){
            return
        }
        if (targetIndex === -1 || currentCardPlacement.status !== 'E') {
            targetIndex = cardPlacements.value.findIndex((card, idx) => idx >= currentCardIndex && card.status === 'E');
        }

        if (targetIndex !== -1) {
            placementsToUpdate.push({
                ...currentCardPlacement,
                id: targetIndex,
                status: status,
                cardName: cardName,
                isFault: isFault,  
            });
            currentCardIndex = targetIndex + 1; // 次の空マスを探すための開始インデックスを更新
        } else {
            console.warn(`マスが全て埋まっているため、札名 '${cardName}' は配置できませんでした。`);
            alert(`マスが全て埋まっているため、札名 '${cardName}' は配置できませんでした。`);
        }
    });

    // 実際の更新
    if (placementsToUpdate.length > 0) {
        cardPlacements.value = cardPlacements.value.map(card => {
            
            const updated = placementsToUpdate.find(p => p.id === card.id);
            return updated ? updated : card;

        });
    }

    // モーダルを閉じる
    showInputModal.value = false;
    editingCardId.value = null; // リセット
};


/**
 * 全て入力完了後の「保存」ボタン処理
 */
const saveMatchRecord = () => {
    const placedCards = cardPlacements.value.filter(card => card.status !== 'E');

    if (placedCards.length === 0) {
        alert('札が一つも配置されていません。');
        return;
    }

    // 試合記録オブジェクトを作成
    const matchRecord = {
        date: new Date().toISOString().slice(0, 10),
        placedCards: placedCards,
        selfScore: placedCards.filter(c => c.acquiredBy === 'Self' && !c.isFault).length,
        opponentScore: placedCards.filter(c => c.acquiredBy === 'Opponent' && !c.isFault).length,
        faultCount: placedCards.filter(c => c.isFault).length,
    };

    savedMatches.value.push(matchRecord);
    
    // グリッドをリセット
    cardPlacements.value = createInitialCardPlacements();
    
    alert(`試合記録を保存しました。\n自取: ${matchRecord.selfScore}, 敵取: ${matchRecord.opponentScore}, お手つき: ${matchRecord.faultCount}`);
    
    console.log('保存された試合記録:', matchRecord);
};

/**
 * 札配置グリッドのセルに適用するクラスを返す (色とサイズを決定)
 */
const getCardCellClasses = (card:typeof cardPlacements["value"][number]) => {
    let bgColor = 'bg-white hover:bg-gray-100';
    let textColor = 'text-gray-400';
    let ring = '';
    let baseTextSize = 'text-xs';
    let displayFlex = 'flex items-center justify-center'; // 常に中央揃え

    if (card.status === 'E') {
        // 札が配置されていない場合 (空): プラスボタンのデザイン
        bgColor = 'bg-gray-100 hover:bg-red-200'; // クリック可能なようにホバー色を赤に変更
        textColor = 'text-gray-500';
        baseTextSize = 'text-xl font-medium'; // プラス記号を大きくする
    } else if (card.acquiredBy === 'Self') {
        // 自分が取った札 (濃い青)
        bgColor = 'bg-blue-500 hover:bg-blue-600';
        textColor = 'text-white'; // マス全体の文字色は白だが、札名自体は黒にする
        baseTextSize = 'text-sm font-semibold';
    } else if (card.acquiredBy === 'Opponent') {
        // 相手が取った札 (薄い黄色/オレンジ)
        bgColor = 'bg-yellow-300 hover:bg-yellow-400';
        textColor = 'text-gray-900'; // マス全体の文字色は黒だが、札名自体は赤にする
        baseTextSize = 'text-sm font-semibold';
    } else if (card.isFault) {
        // お手つき (赤枠)
        bgColor = 'bg-red-50';
        textColor = 'text-red-700'; // マス全体の文字色は赤
        ring = 'ring-2 ring-red-500';
        baseTextSize = 'text-sm font-semibold';
    }

    // w-[6.25%] = 100% / 16列 (レスポンシブ対応のための固定幅)
    return `flex-shrink-0 w-[6.25%] h-12 border border-gray-200 ${displayFlex} rounded-sm transition-all duration-150 relative ${bgColor} ${textColor} ${ring} active:scale-95 cursor-pointer select-none ${baseTextSize} p-1`;
};

/**
 * 札名に適用する文字色クラスを返す
 */
const getCardNameTextColor = (card:typeof cardPlacements["value"][number]) => {
    if (card.acquiredBy === 'Self' && !card.isFault) {
        // 自取: 黒文字
        return 'text-gray-900'; 
    } else if (card.acquiredBy === 'Opponent' && !card.isFault) {
        // 敵取: 赤文字
        return 'text-red-700';
    } else if (card.isFault) {
        // お手つき: 赤文字
        return 'text-red-700';
    }
    // その他の状態（空など）
    return '';
};
</script>

<template>
<h1>記録</h1>
<p>練習日</p>
<UInput type="date" />
<p>第何試合目</p>
<UInput placeholder="第何試合目かを入力(半角数字)" v-model="matchnumberValue" />
<p>記録の種類</p>
<URadioGroup v-model="recordvalue":items="recordItems" />
<p>対戦相手の級・段位</p>
<USelectMenu v-model="gradeValue":items="gradeItems" />
<p>対戦相手の名前</p>
<UInput placeholder="名前を入力" v-model="nameValue":items="nameItmes" />
<p>結果</p>
<URadioGroup v-model="winorlossValue":items="winorlossItems" />
<p>枚数差</p>
<UInput placeholder="枚数差を入力(半角数字)" v-model="numberValue" />

<div class="bg-white p-4 rounded-xl shadow-xl border border-gray-200">
        <h3 class="text-lg font-medium text-gray-700 mb-3">札配置と取得結果 (6段 x 16列) </h3>
        
        <!-- 札配置・結果入力グリッド (6段x16列) --><div class="overflow-x-auto rounded-xl border border-gray-300 shadow-inner">
            <div class="min-w-[960px] flex flex-col">
                <!-- 行 (6段) のループ --><div v-for="row in 6" :key="row" class="flex flex-row w-full">
                    <!-- 列 (16列) のループ --><div
                        v-for="card in cardPlacements.slice((row - 1) * 16, row * 16)"
                        :key="card.id"
                        :class="getCardCellClasses(card)"
                        @click="handleCardClick(card.id)"
                    >
                        <!-- 札が空の場合 (status === 'E') は '+' を表示 --><span v-if="card.status === 'E'" class="text-xl font-medium leading-none flex items-center justify-center">
                            +
                        </span>
                        
                        <!-- 札が配置されている場合: 札名と取得者を表示 --><div v-else class="flex flex-col items-center justify-center h-full w-full">
                            <!-- 札名 --><span class="font-bold text-base leading-none" :class="getCardNameTextColor(card)">
                                 {{ card.cardName }}
                            </span>
                            <!-- 取得者/状態ラベル --><span 
                                :class="{
                                    'text-xs font-medium mt-0.5 px-1 py-0 bg-blue-700 text-white rounded-full leading-none': card.acquiredBy === 'Self' && !card.isFault,
                                    'text-xs font-medium mt-0.5 px-1 py-0 bg-yellow-600 text-gray-900 rounded-full leading-none': card.acquiredBy === 'Opponent' && !card.isFault,
                                    'text-xs font-bold mt-0.5 px-1 py-0 bg-red-800 text-white rounded-full leading-none': card.isFault,
                                }"
                                class="text-xs font-medium mt-0.5 px-1 py-0 rounded-full leading-none"
                            >
                                <template v-if="card.isFault">お手つき</template>
                                <template v-else-if="card.acquiredBy === 'Self'">自取</template>
                                <template v-else-if="card.acquiredBy === 'Opponent'">敵取</template>
                            </span>
                        </div>
                    </div>
                </div>
            </div>
        </div>
        <p class="text-xs text-gray-500 mt-2">
            <span class="font-semibold text-red-600">+ をクリック</span>で札を配置（複数枚入力可）、<span class="font-semibold text-blue-600">配置済みの札をクリック</span>で状態を入力できます。
        </p>

        <!-- 試合記録の保存ボタン --><div class="mt-6 pt-4 border-t border-gray-200">
            <button
                @click="saveMatchRecord"
                class="w-full bg-red-600 hover:bg-red-700 text-white font-semibold py-3 px-6 rounded-xl shadow-lg transition duration-200 focus:outline-none focus:ring-4 focus:ring-red-300 active:bg-red-800"
            >
                入力完了！この配置を試合記録として保存
            </button>
        </div>

        <!-- 札入力モーダル (簡易的なカスタム実装) --><div v-if="showInputModal" class="fixed inset-0 z-50 flex items-center justify-center bg-gray-900 bg-opacity-50 backdrop-blur-sm">
            <div class="bg-white w-full max-w-md p-6 rounded-xl shadow-2xl transform transition-all scale-100">
                <h4 class="text-xl font-bold text-gray-800 mb-4 border-b pb-2">札の配置と取得結果の入力</h4>
                
                <div class="space-y-4">
                    <!-- 1. 札名入力 (複数枚対応) --><div>
                        <label for="cardNames" class="block text-sm font-medium text-gray-700 mb-1">
                            札名 (カンマ区切りで複数枚入力可能)
                            <span class="text-xs text-gray-500">(例: あきの、ちは)</span>
                        </label>
                        <input
                            id="cardNames"
                            v-model="newCardEntry.cardNames"
                            type="text"
                            placeholder="例: あきの、ちは"
                            class="w-full rounded-lg border-gray-300 shadow-sm p-2 border focus:border-red-500 focus:ring-1 focus:ring-red-500"
                        />
                    </div>
                    
                    <!-- 2. 取得者選択 --><div>
                        <label class="block text-sm font-medium text-gray-700 mb-1">取得結果</label>
                        <div class="flex space-x-4 p-2 bg-gray-50 rounded-lg">
                            <label class="flex items-center text-sm">
                                <input type="radio" value="Self" v-model="newCardEntry.acquiredBy" class="mr-1 text-blue-600 focus:ring-blue-500" /> 
                                自取
                            </label>
                            <label class="flex items-center text-sm">
                                <input type="radio" value="Opponent" v-model="newCardEntry.acquiredBy" class="mr-1 text-yellow-600 focus:ring-yellow-500" /> 
                                敵取
                            </label>
                        </div>
                    </div>

                    <!-- 3. お手つきフラグ --><div v-if="newCardEntry.acquiredBy === 'Self' || newCardEntry.acquiredBy === 'Opponent'">
                        <label class="flex items-center text-sm font-medium text-gray-700">
                            <input type="checkbox" v-model="newCardEntry.isFault" class="mr-2 rounded text-red-600 focus:ring-red-500" />
                            お手つき (Fault)
                        </label>
                    </div>
                </div>

                <div class="mt-6 flex justify-end space-x-3">
                    <button
                        @click="showInputModal = false"
                        class="px-4 py-2 text-sm font-medium text-gray-600 bg-gray-100 rounded-lg hover:bg-gray-200 transition"
                    >
                        キャンセル
                    </button>
                    <button
                        @click="saveCardEntry"
                        class="px-4 py-2 text-sm font-medium text-white bg-red-600 rounded-lg hover:bg-red-700 transition shadow-md"
                    >
                        札を配置
                    </button>
                </div>
            </div>
        </div>

    </div>

</template>