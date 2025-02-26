<script setup lang="ts">
import { getAddress } from 'ethers/lib/utils';
import { computed, defineProps, withDefaults } from 'vue';

import useNumbers from '@/composables/useNumbers';
import useTokens from '@/composables/useTokens';
import { PoolToken } from '@/services/balancer/subgraph/types';

import HiddenTokensPills from './HiddenTokensPills.vue';
import StableTokenPill from './StableTokenPill.vue';
import WeightedTokenPill from './WeightedTokenPill.vue';

type Props = {
  tokens: PoolToken[];
  isStablePool?: boolean;
  selectedTokens?: string[];
};

const props = withDefaults(defineProps<Props>(), {
  isStablePool: false,
  selectedTokens: () => []
});

const { fNum2 } = useNumbers();
const { tokens, hasBalance } = useTokens();

/**
 * COMPUTED
 */
const visibleTokens = computed(() => props.tokens.slice(0, MAX_PILLS));

const hiddenTokens = computed(() => props.tokens.slice(MAX_PILLS));

const hasBalanceInHiddenTokens = computed(() =>
  hiddenTokens.value.some(token => hasBalance(token.address))
);

const isSelectedInHiddenTokens = computed(() =>
  hiddenTokens.value.some(token => props.selectedTokens.includes(token.address))
);

/**
 * METHODS
 */
function symbolFor(token: PoolToken): string {
  return (
    token.symbol || tokens.value[getAddress(token.address)]?.symbol || '---'
  );
}

function weightFor(token: PoolToken): string {
  return fNum2(token.weight, {
    style: 'percent',
    maximumFractionDigits: 0
  });
}

const MAX_PILLS = 11;
</script>

<template>
  <div class="-mt-1 flex flex-wrap gap-y-2">
    <template v-if="isStablePool">
      <StableTokenPill
        v-for="token in visibleTokens"
        :key="token.address"
        :hasBalance="hasBalance(token.address)"
        :symbol="symbolFor(token)"
        :token="token"
        :isSelected="selectedTokens.includes(token.address)"
      />
    </template>
    <template v-else>
      <WeightedTokenPill
        v-for="token in visibleTokens"
        :key="token.address"
        :hasBalance="hasBalance(token.address)"
        :symbol="symbolFor(token)"
        :weight="weightFor(token)"
        :token="token"
        :isSelected="selectedTokens.includes(token.address)"
      />
      <HiddenTokensPills
        v-if="hiddenTokens.length > 0"
        :tokens="hiddenTokens"
        :hasBalance="hasBalanceInHiddenTokens"
        :isSelected="isSelectedInHiddenTokens"
      />
    </template>
  </div>
</template>
