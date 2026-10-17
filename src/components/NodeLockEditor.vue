<template>
  <div
    class="fixed inset-0 z-50 flex items-center justify-center bg-black/40"
    @click.self="emit('close')"
  >
    <div
      class="flex flex-col w-[min(60rem,94vw)] max-h-[88vh] bg-gray-50 rounded-xl shadow-2xl"
    >
      <div
        class="flex shrink-0 items-center px-4 py-3 border-b border-gray-300"
      >
        <LockClosedIcon class="w-5 h-5 mr-2 text-gray-700" />
        <div class="text-lg font-bold">
          Node Lock &mdash; {{ capitalize(player) }}
          <span class="ml-2 text-sm font-normal text-gray-600">
            ({{ history.length === 0 ? "root" : history.join(" - ") }})
          </span>
        </div>
        <button
          class="ml-auto opacity-70 hover:opacity-100 text-gray-700"
          @click="emit('close')"
        >
          <XMarkIcon class="w-6 h-6" />
        </button>
      </div>

      <div
        class="flex flex-wrap shrink-0 items-center gap-2 px-4 py-2 border-b border-gray-300 text-sm"
      >
        <button class="button-base button-blue" @click="resetToStrategy">
          Reset to Current Strategy
        </button>
        <button class="button-base button-blue" @click="normalizeAll">
          Normalize All
        </button>

        <div class="ml-auto text-gray-600">
          {{ lockedHandCount }} / {{ rows.length }} locked hand(s)
        </div>

        <button
          class="button-base button-green"
          :disabled="lockedHandCount === 0"
          @click="lockNode"
        >
          Lock Node
        </button>
        <button
          class="button-base button-red"
          :disabled="!isLocked"
          @click="emit('unlock')"
        >
          Clear Lock
        </button>
      </div>

      <div class="flex-grow overflow-auto snug">
        <table class="w-full text-sm text-center align-middle">
          <thead class="sticky top-0 z-20 bg-gray-100 shadow">
            <tr class="h-9">
              <th class="sticky left-0 z-30 bg-gray-100 px-2 min-w-[4.5rem]">
                Hand
              </th>
              <th
                v-for="action in actions"
                :key="action.index"
                class="px-1 min-w-[4rem] whitespace-nowrap"
              >
                {{ actionLabel(action) }}
              </th>
              <th class="px-2 min-w-[3.5rem]">Sum</th>
              <th class="px-2 min-w-[4rem]">Lock</th>
            </tr>
          </thead>

          <tbody>
            <tr
              v-for="row in rows"
              :key="row.hand"
              class="h-8 border-b border-gray-200 bg-gray-50"
            >
              <td class="sticky left-0 z-10 bg-gray-50 px-2">
                <span
                  v-for="card in row.cards"
                  :key="card.rank + card.suit"
                  :class="card.colorClass"
                >
                  {{ card.rank + card.suit }}
                </span>
              </td>

              <td
                v-for="(action, ai) in actions"
                :key="action.index"
                class="px-1"
              >
                <input
                  type="number"
                  min="0"
                  max="100"
                  step="1"
                  class="w-16 px-1 py-0.5 text-right rounded border-gray-400"
                  :value="percentText(row.freqs[ai])"
                  @change="onInput(ai, row.hand, $event)"
                />
              </td>

              <td :class="row.sum > 0 ? '' : 'text-red-600'">
                {{ (row.sum * 100).toFixed(0) }}%
              </td>

              <td>
                <button
                  class="px-1.5 rounded-md transition-colors hover:bg-blue-100"
                  @click="toggleHand(row.hand)"
                >
                  <CheckIcon v-if="row.sum > 0" class="w-4 h-4 text-blue-600" />
                  <span v-else class="text-gray-400">--</span>
                </button>
              </td>
            </tr>
          </tbody>
        </table>
      </div>

      <div
        class="shrink-0 px-4 py-2 text-xs text-gray-600 border-t border-gray-300"
      >
        Frequencies are entered as percentages. Hands with all actions at 0%
        remain unlocked and stay solver-controlled. Locking records the
        constraint and only takes effect after re-solving.
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { computed, defineComponent, ref, watch } from "vue";
import { capitalize, cardText } from "../utils";
import { LockClosedIcon, XMarkIcon, CheckIcon } from "@heroicons/vue/20/solid";

type ActionInfo = { index: number; name: string; amount: string };

type Row = {
  hand: number;
  cards: ReturnType<typeof cardText>[];
  freqs: number[];
  sum: number;
};

export default defineComponent({
  components: {
    LockClosedIcon,
    XMarkIcon,
    CheckIcon,
  },

  props: {
    player: {
      type: String as () => "oop" | "ip",
      required: true,
    },
    history: {
      type: Array as () => number[],
      required: true,
    },
    cards: {
      type: Array as () => number[],
      required: true,
    },
    weights: {
      type: Array as () => number[],
      required: true,
    },
    actions: {
      type: Array as () => ActionInfo[],
      required: true,
    },
    strategy: {
      type: Array as () => number[],
      required: true,
    },
    existingFrequencies: {
      type: Array as () => number[] | null,
      default: null,
    },
    isLocked: {
      type: Boolean,
      required: true,
    },
  },

  emits: {
    lock: (_frequencies: Float32Array) => true,
    unlock: () => true,
    close: () => true,
  },

  setup(props, { emit }) {
    const numActions = computed(() => props.actions.length);
    const numHands = computed(() => props.cards.length);

    const frequencies = ref<number[]>([]);

    const initFrom = (base: number[] | null) => {
      const size = numHands.value * numActions.value;
      const source = base ?? props.strategy;
      const arr = new Array<number>(size).fill(0);
      for (let i = 0; i < size; ++i) {
        arr[i] = Math.max(source[i] ?? 0, 0);
      }
      frequencies.value = arr;
    };

    const initFrequencies = () => {
      initFrom(props.existingFrequencies ?? props.strategy);
    };

    initFrequencies();

    watch(
      () => [props.cards, props.strategy, props.existingFrequencies],
      initFrequencies
    );

    const pairText = (pair: number) => {
      const card1 = pair & 0xff;
      const card2 = pair >>> 8;
      return card2 !== 0xff
        ? [cardText(card2), cardText(card1)]
        : [cardText(card1)];
    };

    const rows = computed<Row[]>(() => {
      const ret: Row[] = [];
      const n = numHands.value;
      const na = numActions.value;

      for (let h = 0; h < n; ++h) {
        if (!(props.weights[h] > 0)) continue;

        const freqs: number[] = [];
        let sum = 0;
        for (let a = 0; a < na; ++a) {
          const value = frequencies.value[a * n + h] ?? 0;
          freqs.push(value);
          sum += value;
        }

        ret.push({ hand: h, cards: pairText(props.cards[h]), freqs, sum });
      }

      return ret;
    });

    const actionLabel = (action: ActionInfo) => {
      if (action.amount === "0") return action.name;
      return `${action.name[0]} ${action.amount}`;
    };

    const percentText = (value: number) => {
      return (Math.max(value, 0) * 100).toFixed(0);
    };

    const setFrequency = (action: number, hand: number, value: number) => {
      const index = action * numHands.value + hand;
      frequencies.value[index] = Math.max(Math.min(value, 1), 0);
    };

    const onInput = (action: number, hand: number, event: Event) => {
      const target = event.target as HTMLInputElement;
      const percent = Number(target.value);
      if (Number.isFinite(percent)) {
        setFrequency(action, hand, percent / 100);
      }
      target.value = percentText(
        frequencies.value[action * numHands.value + hand]
      );
    };

    const normalizeHand = (hand: number) => {
      const n = numHands.value;
      const na = numActions.value;
      let sum = 0;
      for (let a = 0; a < na; ++a) {
        sum += Math.max(frequencies.value[a * n + hand] ?? 0, 0);
      }
      if (sum <= 0) return;
      for (let a = 0; a < na; ++a) {
        frequencies.value[a * n + hand] =
          Math.max(frequencies.value[a * n + hand] ?? 0, 0) / sum;
      }
    };

    const toggleHand = (hand: number) => {
      const n = numHands.value;
      const na = numActions.value;

      let sum = 0;
      for (let a = 0; a < na; ++a) {
        sum += frequencies.value[a * n + hand] ?? 0;
      }

      if (sum > 0) {
        for (let a = 0; a < na; ++a) {
          frequencies.value[a * n + hand] = 0;
        }
      } else {
        let strategySum = 0;
        for (let a = 0; a < na; ++a) {
          strategySum += Math.max(props.strategy[a * n + hand] ?? 0, 0);
        }
        for (let a = 0; a < na; ++a) {
          const fallback = 1 / na;
          frequencies.value[a * n + hand] =
            strategySum > 0
              ? Math.max(props.strategy[a * n + hand] ?? 0, 0) / strategySum
              : fallback;
        }
      }
    };

    const resetToStrategy = () => {
      initFrom(props.strategy);
    };

    const normalizeAll = () => {
      for (const row of rows.value) {
        normalizeHand(row.hand);
      }
    };

    const lockNode = () => {
      const size = numHands.value * numActions.value;
      const arr = new Float32Array(size);
      for (let i = 0; i < size; ++i) {
        arr[i] = Math.max(frequencies.value[i] ?? 0, 0);
      }
      emit("lock", arr);
    };

    const lockedHandCount = computed(
      () => rows.value.filter((row) => row.sum > 0).length
    );

    return {
      capitalize,
      frequencies,
      rows,
      actionLabel,
      percentText,
      onInput,
      normalizeHand,
      toggleHand,
      resetToStrategy,
      normalizeAll,
      lockNode,
      lockedHandCount,
    };
  },
});
</script>
