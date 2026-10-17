<template>
  <div v-if="!store.isSolverFinished">
    <div class="flex w-full max-w-screen-xl mx-auto px-4 py-6 items-center">
      <span
        v-if="store.isSolverRunning || store.isFinalizing"
        class="spinner inline-block mr-3"
      ></span>
      {{
        !store.hasSolverRun
          ? "Solver has not run."
          : store.isSolverRunning
          ? "Solver running..."
          : store.isFinalizing
          ? "Finalizing..."
          : "Solver paused."
      }}
    </div>
  </div>

  <div v-else class="flex flex-col h-full">
    <ResultNav
      :is-handler-updated="isHandlerUpdated"
      :is-locked="isLocked"
      :cards="cards"
      :dealt-card="dealtCard"
      @update:is-handler-updated="(value) => (isHandlerUpdated = value)"
      @update:is-locked="(value) => (isLocked = value)"
      @trigger-update="onUpdateSpot"
    />

    <ResultMiddle
      :display-mode="displayMode"
      :chance-mode="chanceMode"
      :auto-player-basics="autoPlayerBasics"
      :auto-player-chance="autoPlayerChance"
      :copy-success="copySuccess"
      :can-lock="canLock"
      :is-node-locked="isNodeLocked"
      :lock-count="lockCount"
      @update:display-mode="updateDisplayMode"
      @update:display-options="updateDisplayOptions"
      @copy-to-clipboard="copyRangeTextToClipboard"
      @reset-copy-success="resetCopySuccess"
      @open-lock-editor="lockEditorOpen = true"
      @re-solve-with-locks="store.requestReSolve()"
    />

    <div
      v-if="store.navView === 'results' && selectedSpot && results"
      class="flex flex-grow min-h-0"
    >
      <template v-if="displayMode === 'basics'">
        <ResultBasics
          style="flex: 4"
          :cards="cards"
          :selected-spot="selectedSpot"
          :selected-chance="selectedChance"
          :current-board="currentBoard"
          :total-bet-amount="totalBetAmount"
          :results="results"
          :display-options="displayOptions"
          :display-player="displayPlayerBasics"
          :is-compare-mode="false"
          @update-hover-content="onUpdateHoverContent"
        />

        <ResultTable
          style="flex: 3"
          table-mode="basics"
          :cards="cards"
          :selected-spot="selectedSpot"
          :results="results"
          :display-player="displayPlayerBasics"
          :hover-content="basicsHoverContent"
        />
      </template>

      <template v-else-if="displayMode === 'graphs'">
        <ResultGraphs
          :cards="cards"
          :selected-spot="selectedSpot"
          :selected-chance="selectedChance"
          :results="results"
          :chance-reports="chanceReports"
          :display-options="displayOptions"
          :display-player="displayPlayerBasics"
        />
      </template>

      <template v-else-if="displayMode === 'compare'">
        <ResultBasics
          style="flex: 5"
          :cards="cards"
          :selected-spot="selectedSpot"
          :selected-chance="selectedChance"
          :current-board="currentBoard"
          :total-bet-amount="totalBetAmount"
          :results="results"
          :display-options="displayOptions"
          display-player="oop"
          :is-compare-mode="true"
        />

        <ResultCompare
          style="flex: 2"
          :selected-spot="selectedSpot"
          :selected-chance="selectedChance"
          :results="results"
        />

        <ResultBasics
          style="flex: 5"
          :cards="cards"
          :selected-spot="selectedSpot"
          :selected-chance="selectedChance"
          :current-board="currentBoard"
          :total-bet-amount="totalBetAmount"
          :results="results"
          :display-options="displayOptions"
          display-player="ip"
          :is-compare-mode="true"
        />
      </template>

      <template v-else-if="displayMode === 'chance' && selectedChance">
        <ResultChance
          :selected-spot="selectedSpot"
          :selected-chance="selectedChance"
          :chance-reports="chanceReports"
          :display-options="displayOptions"
          :display-player="displayPlayerChance"
          @deal-card="onDealCard"
        />
      </template>
    </div>

    <NodeLockEditor
      v-if="lockEditorOpen && lockEditorContext"
      :player="lockEditorContext.player"
      :history="lockEditorContext.history"
      :cards="lockEditorContext.cards"
      :weights="lockEditorContext.weights"
      :actions="lockEditorContext.actions"
      :strategy="lockEditorContext.strategy"
      :existing-frequencies="existingFrequencies"
      :is-locked="isNodeLocked"
      @lock="onLockNode"
      @unlock="onUnlockNode"
      @close="lockEditorOpen = false"
    />

    <div
      v-if="lockError"
      class="fixed bottom-4 right-4 z-50 px-4 py-2 bg-red-600 text-white text-sm rounded-lg shadow-lg"
    >
      {{ lockError }}
    </div>
  </div>
</template>

<script lang="ts">
import { computed, defineComponent, ref } from "vue";
import { useStore, useLockedNodesStore } from "../store";
import { handler } from "../global-worker";

import {
  Results,
  ChanceReports,
  Spot,
  SpotChance,
  SpotPlayer,
  DisplayMode,
  DisplayOptions,
  HoverContent,
  LockedNode,
} from "../result-types";

import ResultNav from "./ResultNav.vue";
import ResultMiddle from "./ResultMiddle.vue";
import ResultBasics from "./ResultBasics.vue";
import ResultTable from "./ResultTable.vue";
import ResultCompare from "./ResultCompare.vue";
import ResultGraphs from "./ResultGraphs.vue";
import ResultChance from "./ResultChance.vue";
import NodeLockEditor from "./NodeLockEditor.vue";

export default defineComponent({
  components: {
    ResultNav,
    ResultMiddle,
    ResultBasics,
    ResultTable,
    ResultCompare,
    ResultGraphs,
    ResultChance,
    NodeLockEditor,
  },

  setup() {
    const store = useStore();
    const lockedNodes = useLockedNodesStore();

    /* Navigation */

    const isHandlerUpdated = ref(false);
    const isLocked = ref(false);

    const cards = ref<number[][]>([[], []]);
    const dealtCard = ref(-1);

    const selectedSpot = ref<Spot | null>(null);
    const selectedChance = ref<SpotChance | null>(null);
    const selectedHistory = ref<number[]>([]);
    const currentBoard = ref<number[]>([]);
    const results = ref<Results | null>(null);
    const chanceReports = ref<ChanceReports | null>(null);
    const totalBetAmount = ref([0, 0]);

    const lockEditorOpen = ref(false);

    const isSolverFinished = ref(false);
    store.$subscribe(async (_, store) => {
      if (isSolverFinished.value !== store.isSolverFinished) {
        if ((isSolverFinished.value = store.isSolverFinished)) {
          await init();
        } else {
          clear();
        }
      }
    });

    const init = async () => {
      if (!handler) return;

      const cardsBuffer = [
        await handler.privateCards(0),
        await handler.privateCards(1),
      ];

      cards.value = Array.from({ length: 2 }, (_, player) => {
        return Array.from(cardsBuffer[player]);
      });

      isHandlerUpdated.value = true;
    };

    const clear = () => {
      cards.value = [[], []];
      selectedSpot.value = null;
      selectedChance.value = null;
      results.value = null;
      chanceReports.value = null;
    };

    const onUpdateSpot = (
      newSelectedSpot: Spot | null,
      newSelectedChance: SpotChance | null,
      newCurrentBoard: number[],
      newResults: Results,
      newChanceReports: ChanceReports | null,
      newTotalBetAmount: number[],
      newHistory: number[]
    ) => {
      dealtCard.value = -1;
      selectedSpot.value = newSelectedSpot;
      selectedChance.value = newSelectedChance;
      currentBoard.value = newCurrentBoard;
      results.value = newResults;
      chanceReports.value = newChanceReports;
      totalBetAmount.value = newTotalBetAmount;
      selectedHistory.value = newHistory;
      isLocked.value = false;
      lockEditorOpen.value = false;

      chanceMode.value = newSelectedChance?.player ?? "";
    };

    /* Middle Bar */

    const displayMode = ref<DisplayMode>("basics");
    const chanceMode = ref("");

    const displayOptions = ref<DisplayOptions>({
      playerBasics: "auto",
      playerChance: "auto",
      barHeight: "normalized",
      suit: "grouped",
      strategy: "show",
      contentBasics: "default",
      contentGraphs: "eq",
      chartChance: "strategy-combos",
    });

    const copySuccess = ref(0);

    const updateDisplayMode = (mode: DisplayMode) => {
      displayMode.value = mode;
    };

    const updateDisplayOptions = (options: DisplayOptions) => {
      displayOptions.value = options;
    };

    const copyRangeTextToClipboard = async () => {
      const text = "Hello World";
      navigator.clipboard
        .writeText(text)
        .then(() => (copySuccess.value = 1))
        .catch(() => (copySuccess.value = -1));
    };

    const resetCopySuccess = () => {
      copySuccess.value = 0;
    };

    /* Computed */

    const autoPlayerBasics = computed(() => {
      const spot = selectedSpot.value;
      const chance = selectedChance.value;
      if (!spot) return "oop";

      if (chance) {
        return chance.prevPlayer;
      } else if (spot.type === "terminal") {
        return spot.prevPlayer;
      } else {
        return (spot as SpotPlayer).player;
      }
    });

    const autoPlayerChance = computed(() => {
      const spot = selectedSpot.value;
      if (!spot) return "oop";
      if (spot.type === "terminal") {
        return spot.prevPlayer;
      } else {
        return (spot as SpotPlayer).player;
      }
    });

    const displayPlayerBasics = computed(() => {
      const optionPlayer = displayOptions.value.playerBasics;
      if (optionPlayer === "auto") {
        return autoPlayerBasics.value;
      } else {
        return optionPlayer;
      }
    });

    const displayPlayerChance = computed(() => {
      const optionPlayer = displayOptions.value.playerChance;
      if (optionPlayer === "auto") {
        return autoPlayerChance.value;
      } else {
        return optionPlayer;
      }
    });

    /* Results */

    const basicsHoverContent = ref<HoverContent | null>(null);

    const onUpdateHoverContent = (content: HoverContent | null) => {
      basicsHoverContent.value = content;
    };

    const onDealCard = (card: number) => {
      dealtCard.value = card;
    };

    /* Node Locking */

    const lockError = ref("");

    type LockEditorContext = {
      player: "oop" | "ip";
      history: number[];
      cards: number[];
      weights: number[];
      actions: { index: number; name: string; amount: string }[];
      strategy: number[];
    };

    const lockEditorContext = computed<LockEditorContext | null>(() => {
      const spot = selectedSpot.value;
      const res = results.value;
      if (!spot || spot.type !== "player" || !res) return null;
      if (res.numActions <= 0 || res.isEmpty || res.strategy.length === 0) {
        return null;
      }

      const history = selectedHistory.value;
      if (history.some((index) => index < 0)) return null;

      const playerIndex = spot.player === "oop" ? 0 : 1;
      if (res.weights[playerIndex].length === 0) return null;

      return {
        player: spot.player,
        history,
        cards: cards.value[playerIndex],
        weights: res.weights[playerIndex],
        actions: spot.actions.map((action) => ({
          index: action.index,
          name: action.name,
          amount: action.amount,
        })),
        strategy: res.strategy,
      };
    });

    const existingFrequencies = computed<number[] | null>(() => {
      const context = lockEditorContext.value;
      if (!context) return null;
      const lock = lockedNodes.getLock(context.history);
      return lock ? lock.frequencies : null;
    });

    const isNodeLocked = computed(() => {
      const context = lockEditorContext.value;
      return context ? lockedNodes.hasLock(context.history) : false;
    });

    const canLock = computed(() => lockEditorContext.value !== null);

    const lockCount = computed(() => lockedNodes.count);

    const onLockNode = (frequencies: Float32Array) => {
      const context = lockEditorContext.value;
      if (!context) return;

      lockError.value = "";

      // Locking is a pre-solve constraint: it cannot be applied while the game
      // is solved. Record the lock in the store and let the next re-solve apply
      // it (see `applyStoredLocks` in RunSolver.vue).
      const expectedLength = context.actions.length * context.cards.length;
      if (frequencies.length !== expectedLength) {
        lockError.value = `Invalid strategy length: expected ${expectedLength}, got ${frequencies.length}`;
        return;
      }

      const lock: LockedNode = {
        history: context.history,
        player: context.player,
        numActions: context.actions.length,
        numHands: context.cards.length,
        frequencies: Array.from(frequencies),
      };
      lockedNodes.setLock(lock);
      lockEditorOpen.value = false;
    };

    const onUnlockNode = () => {
      const context = lockEditorContext.value;
      if (!context) return;

      lockError.value = "";

      // The solver applies the removal on the next re-solve, where stale locks
      // are explicitly unlocked after `allocateMemory`.
      lockedNodes.removeLock(context.history);
      lockEditorOpen.value = false;
    };

    return {
      store,
      isHandlerUpdated,
      isLocked,
      cards,
      dealtCard,
      selectedSpot,
      selectedChance,
      currentBoard,
      results,
      chanceReports,
      totalBetAmount,
      onUpdateSpot,
      displayMode,
      chanceMode,
      displayOptions,
      updateDisplayMode,
      updateDisplayOptions,
      copySuccess,
      copyRangeTextToClipboard,
      resetCopySuccess,
      autoPlayerBasics,
      autoPlayerChance,
      displayPlayerBasics,
      displayPlayerChance,
      basicsHoverContent,
      onUpdateHoverContent,
      onDealCard,
      lockEditorOpen,
      lockEditorContext,
      existingFrequencies,
      isNodeLocked,
      canLock,
      lockCount,
      onLockNode,
      onUnlockNode,
      lockError,
    };
  },
});
</script>
