<script setup lang="ts">
/**
 * A series of actions the user must perform, displayed horizontally as a series of dots
 * As each action is in progress or completed the dot changes to reflect its
 * current state.
 *
 * Useful if there are an arbitrary number of actions the user must take such as
 * "approve n tokens, then invest in a pool.""
 */
import { ref, computed, reactive } from 'vue';
import {
  TransactionReceipt,
  TransactionResponse
} from '@ethersproject/abstract-provider';
import { Step, StepState } from '@/types';
import {
  TransactionAction,
  TransactionActionInfo,
  TransactionActionState
} from '@/types/transactions';
import useEthers from '@/composables/useEthers';
import { dateTimeLabelFor } from '@/composables/useTime';
import useTransactionErrors from '@/composables/useTransactionErrors';

/**
 * TYPES
 */
type Props = {
  actions: TransactionActionInfo[];
};

/**
 * PROPS & EMITS
 */
const props = defineProps<Props>();

const emit = defineEmits<{
  (e: 'success', value: any): void;
}>();

const defaultActionState: TransactionActionState = {
  init: false,
  confirming: false,
  confirmed: false,
  confirmedAt: ''
};

/**
 * STATE
 */
const currentActionIndex = ref(0);

const actionStates = props.actions.map(() => {
  return reactive<TransactionActionState>({
    ...defaultActionState
  });
});

/**
 * COMPOSABLES
 */
const { txListener, getTxConfirmedAt } = useEthers();
const { parseError } = useTransactionErrors();

/**
 * COMPUTED
 */

const actions = computed((): TransactionAction[] => {
  return props.actions.map((actionInfo, idx) => {
    const actionState = actionStates[idx];
    return {
      label: actionInfo.label,
      loadingLabel: actionState.init
        ? actionInfo.loadingLabel
        : actionInfo.confirmingLabel,
      pending: actionState.init || actionState.confirming,
      promise: submit.bind(null, actionInfo.action, actionState),
      step: {
        tooltip: actionInfo.stepTooltip,
        state: getStepState(actionState, idx)
      }
    };
  });
});

const currentAction = computed(
  (): TransactionAction => actions.value[currentActionIndex.value]
);

const currentActionState = computed(
  (): TransactionActionState => actionStates[currentActionIndex.value]
);

const lastActionState = computed(
  (): TransactionActionState => actionStates[actionStates.length - 1]
);

const steps = computed((): Step[] => actions.value.map(action => action.step));

const spacerWidth = computed((): number => {
  return 13 - steps.value.length;
});

/**
 * METHODS
 */

function getStepState(
  actionState: TransactionActionState,
  index: number
): StepState {
  if (currentActionIndex.value < index) return StepState.Todo;
  else if (actionState.confirming) return StepState.Pending;
  else if (actionState.init) return StepState.WalletOpen;
  else if (actionState.confirmed) return StepState.Success;
  return StepState.Active;
}

async function submit(
  action: () => Promise<TransactionResponse>,
  state: TransactionActionState
): Promise<void> {
  try {
    state.init = true;
    state.error = null;

    const tx = await action();

    state.init = false;
    state.confirming = true;

    handleTransaction(tx, state);
  } catch (error) {
    state.init = false;
    state.confirming = false;
    state.error = parseError(error);
    console.error(error);
  }
}

async function handleTransaction(
  tx: TransactionResponse,
  state: TransactionActionState
): Promise<void> {
  state.confirmed = await txListener(tx, {
    onTxConfirmed: async (receipt: TransactionReceipt) => {
      state.receipt = receipt;

      const confirmedAt = await getTxConfirmedAt(receipt);
      state.confirmedAt = dateTimeLabelFor(confirmedAt);

      if (currentActionIndex.value >= actions.value.length - 1) {
        emit('success', { receipt, confirmedAt: state.confirmedAt });
      } else {
        currentActionIndex.value += 1;
      }

      state.confirming = false;
    },
    onTxFailed: () => {
      state.confirming = false;
    }
  });
}
</script>

<template>
  <div>
    <BalAlert
      v-if="currentActionState.error"
      type="error"
      :title="currentActionState.error.title"
      :description="currentActionState.error.description"
      block
      class="mb-4"
    />
    <BalHorizSteps
      v-if="actions.length > 1 && !lastActionState.confirmed"
      :steps="steps"
      :spacerWidth="spacerWidth"
      class="flex justify-center"
    />
    <BalBtn
      v-if="!lastActionState.confirmed"
      color="gradient"
      class="mt-4"
      :loading="currentAction.pending"
      :loading-label="currentAction.loadingLabel"
      block
      @click="currentAction.promise()"
    >
      {{ currentAction.label }}
    </BalBtn>
  </div>
</template>
