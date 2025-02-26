<template>
  <BalCard
    class="overflow-x-auto whitespace-nowrap"
    :square="upToLargeBreakpoint"
    :noBorder="upToLargeBreakpoint"
    noPad
  >
    <BalTable
      :columns="columns"
      :data="tableData"
      :is-loading="loading"
      skeleton-class="h-64"
      sticky="both"
      :initial-state="{
        sortColumn: 'weight',
        sortDirection: 'desc'
      }"
    >
      <template v-slot:tokenColumnCell="token">
        <div class="px-6 py-4 flex flex-row w-52">
          <BalLink
            :href="explorer.addressLink(token.address)"
            external
            noStyle
            class="flex items-center"
          >
            <BalAsset :address="token.address" :size="36" />
            <span class="pl-4 font-medium eth-address">
              {{ symbolFor(token.address) }}
            </span>
            <BalIcon
              name="arrow-up-right"
              size="sm"
              class="ml-2 text-gray-500 hover:text-blue-500 transition-colors"
            />
          </BalLink>
        </div>
      </template>
      <template v-slot:tokenWeightCell="token">
        <div class="px-6 py-4 text-right font-numeric">
          {{ weightFor(token.address) }}
        </div>
      </template>
      <template v-slot:tokenBalanceCell="token">
        <div class="px-6 py-4 text-right font-numeric">
          {{ balanceFor(token.address) }}
        </div>
      </template>
      <template v-slot:tokenValueCell="token">
        <div class="px-6 py-4 text-right font-numeric">
          {{ fiatValueFor(token.address) }}
        </div>
      </template>
    </BalTable>
  </BalCard>
</template>

<script lang="ts">
import numeral from 'numeral';
import { computed, defineComponent, PropType, Ref, toRefs } from 'vue';
import { useI18n } from 'vue-i18n';

import useBreakpoints from '@/composables/useBreakpoints';
import useNumbers, { FNumFormats } from '@/composables/useNumbers';
import { usePool } from '@/composables/usePool';
import useTokens from '@/composables/useTokens';
import { shortenLabel } from '@/lib/utils';
import { FullPool } from '@/services/balancer/subgraph/types';
import useWeb3 from '@/services/web3/useWeb3';

export default defineComponent({
  props: {
    pool: {
      type: Object as PropType<FullPool>,
      required: true
    },
    loading: { type: Boolean, default: false }
  },
  setup(props) {
    /**
     * STATE
     */
    const { pool }: { pool: Ref<FullPool> } = toRefs(props);

    /**
     * COMPOSABLES
     */
    const { fNum2 } = useNumbers();
    const { explorerLinks } = useWeb3();
    const { t } = useI18n();
    const { upToLargeBreakpoint } = useBreakpoints();
    const { priceFor } = useTokens();
    const { isStableLikePool } = usePool(pool);

    /**
     * COMPUTED
     */
    const tableData = computed(() => {
      if (!pool || !pool.value || props.loading) return [];
      return Object.keys(pool.value.onchain.tokens).map((address, index) => ({
        address,
        index
      }));
    });

    const columns = computed(() => [
      {
        name: t('token'),
        id: 'token',
        accessor: 'address',
        Cell: 'tokenColumnCell',
        width: 175
      },
      {
        name: t('weight'),
        id: 'weight',
        accessor: 'index',
        Cell: 'tokenWeightCell',
        align: 'right',
        sortKey: pool => weightFor(pool.address),
        width: 125,
        hidden: !props.loading && isStableLikePool.value
      },
      {
        name: t('balance'),
        id: 'balance',
        accessor: 'index',
        Cell: 'tokenBalanceCell',
        align: 'right',
        sortKey: pool => balanceFor(pool.address),
        width: 125
      },
      {
        name: t('value'),
        id: 'value',
        accessor: 'index',
        Cell: 'tokenValueCell',
        align: 'right',
        sortKey: pool => numeral(fiatValueFor(pool.address)).value(),
        width: 125
      }
    ]);

    /**
     * METHODS
     */
    function symbolFor(address: string) {
      if (!pool || !pool.value) return '-';
      const symbol = pool.value.onchain.tokens[address].symbol;
      return symbol ? symbol : shortenLabel(address);
    }

    function balanceFor(address: string): string {
      if (!pool || !pool.value) return '-';
      return fNum2(
        pool.value.onchain.tokens[address].balance,
        FNumFormats.token
      );
    }

    function weightFor(address: string): string {
      if (!pool || !pool.value) return '-';
      return fNum2(
        pool.value.onchain.tokens[address].weight,
        FNumFormats.percent
      );
    }

    function fiatValueFor(address: string): string {
      const price = priceFor(address);
      if (!pool || !pool.value || price === 0) return '-';

      const balance = Number(pool.value.onchain.tokens[address].balance);
      return fNum2(balance * price, FNumFormats.fiat);
    }

    return {
      symbolFor,
      balanceFor,
      weightFor,
      fiatValueFor,
      fNum2,
      explorer: explorerLinks,
      columns,
      tableData,
      upToLargeBreakpoint
    };
  }
});
</script>
