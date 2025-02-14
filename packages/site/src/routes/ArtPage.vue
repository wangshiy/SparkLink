<template>
  <div v-if="!isCurrentChain" style="margin-top: 40px">
    <el-alert type="error" dark>
      {{ t("hint", [chain, web3InfoGetter.chain.name.value]) }}
    </el-alert>
  </div>
  <template v-else-if="metadata">
    <div class="art-detail">
      <img
        src="/assets/art/expand-left.png"
        class="back-to-collection"
        @click="router.back()"
      />
      <h2 class="title">{{ t("title") }}</h2>
      <div class="current-earnings" v-if="owned">
        <p class="note">{{ t("current_earning") }}</p>
        <div class="result">
          <p class="number">{{ toCoin(metadata.profit) }}</p>
          <p class="unit">{{ token.symbol }}</p>
        </div>
        <div class="btns">
          <button
            class="btn orange receive"
            @click="showClaimInProgress = true"
          >
            {{ t("receive_income") }}
          </button>
          <button class="btn orange disabled">{{ t("transfer") }}</button>
        </div>
      </div>
      <div class="detail-container">
        <div class="cover">
          <div class="cover-container">
            <img :src="metadata.urls.cover" />
          </div>
          <h2 class="ntf-name">{{ metadata.name }}</h2>
          <p class="description">{{ metadata.description }}</p>
          <Operations
            style="text-align: left"
            :metadata="metadata"
            :nft-id="nftId"
          ></Operations>
        </div>
        <div class="info">
          <p class="ntf-id">#{{ nftId }}</p>
          <div class="nft-detail">
            <p>
              {{
                t("detail.price", {
                  price: sellingPriceInCoin,
                  symbol: token.symbol,
                })
              }}
            </p>
            <p>
              {{
                t("detail.royalty", {
                  price: toCoin(metadata.royaltyPrice),
                  symbol: token.symbol,
                })
              }}
            </p>
            <p>
              {{
                t("detail.max_shares_number", { times: metadata.maxShareTimes })
              }}
            </p>
            <p>
              {{
                t("detail.percentage_earning", {
                  percentage: metadata.percentageOfEarnings,
                })
              }}
            </p>
            <p>
              {{
                t("detail.work_is_encrypted", { encrypted: metadata.encrypted })
              }}
            </p>
            <p>
              {{
                t("detail.allow_secondary_creation", {
                  allow: metadata.allowSecondaryCreation,
                })
              }}
            </p>
            <p>{{ t("detail.child_node_owned", {}) }}</p>
          </div>
          <div>
            <button class="btn white" @click="copyLink">
              {{ t("share.link") }}
            </button>
            <button class="btn white" @click="showPoster = true">
              {{ t("share.poster") }}
            </button>
          </div>
          <p class="notes">
            {{ t("notes") }}
          </p>
        </div>
      </div>
    </div>
    <Poster
      v-model="showPoster"
      :cover="metadata.urls.cover"
      :name="metadata.name"
      :price="metadata.sellingPrice"
      :symbol="token.symbol"
      :logo-url="token.logoURI"
      :chain="chain"
      :payment-currency="metadata.paymentCurrency.value"
      :url="shareLink"
    ></Poster>
    <ClaimProfitInProgress
      v-if="showClaimInProgress"
      v-model="showClaimInProgress"
      :nft-id="nftId"
    ></ClaimProfitInProgress>
  </template>
  <el-skeleton v-else :rows="5" animated style="margin-top: 40px" />
</template>

<script lang="ts" setup>
import { computed, onMounted, ref, watch } from "vue";
import { useRoute, useRouter } from "vue-router";
import { useStore } from "vuex";
import { UserOperatorFactory, DownloadEventEmitter } from "@SparkLink/business";
import type { INftInformation } from "@SparkLink/business/generated/src/nftInfomation";
import type { IContentDownloader } from "@SparkLink/business/generated/src/downloader";
import Poster from "../components/art/Poster.vue";
import { ElMessage } from "element-plus";
import { web3InfoGetter } from "../store";
import { chainIdToName, IToken } from "../token";
import { useI18n } from "vue-i18n";
import type Web3 from "web3";
import Operations from "../components/art/Operations.vue";
import ClaimProfitInProgress from "../components/art/ClaimProfitInProgress.vue";

function g() {
  console.log(chain, web3InfoGetter.chain.name);
}

const { t } = useI18n({
  messages: {
    en: {
      title: "Collection",
      current_earning: "Current Earnings",
      receive_income: "RECEIVE INCOME",
      transfer: "TRANSFER",
      hint: "This nft is on the {0} network. Please switch to it first. (Current: {1})",
      detail: {
        price: "Spark Price: {price} {symbol}",
        royalty: "Fixed royalty for root: {price} {symbol}",
        max_shares_number: "Maximum number of share: {times}",
        percentage_earning: "Percentage of Earning on sub-node: {percentage}%",
        work_is_encrypted: "Work is encrypted: {encrypted}",
        allow_secondary_creation: "Allow secondary creation: {allow}",
        child_node_owned: "Number of child nodes currently owned:  0",
      },
      share: {
        link: "LINK SHARING",
        poster: "POSTER SHARING",
      },
      copyLink: {
        hint: "The link has been copyed to your clipboard.",
        prefix: "You are welcome to view and try casting my NFT publication: ",
      },
      notes:
        "💡Everyone can view and purchase this work by sharing links or posters to the details page of the work.",
    },
    "zh-CN": {
      title: "收藏",
      current_earning: "目前收益",
      receive_income: "获取收益",
      transfer: "转移",
      hint: "当前NFT在 {0} 网络。请先更换到该网络。（当前网络：{1}）",
      detail: {
        price: "点火价格：{price} {symbol}",
        royalty: "根节点固定版税抽成：{price} {symbol}",
        max_shares_number: "最大分销次数：{times}",
        percentage_earning: "子节点抽成: {percentage}%",
        work_is_encrypted: "NFT内容是否加密{encrypted}",
        allow_secondary_creation: "是否允许二次创作：{allow}",
        child_node_owned: "目前已经有的下级节点：0",
      },
      share: {
        link: "复制链接",
        poster: "分享海报",
      },
      copyLink: {
        hint: "链接已被复制到剪切板。",
        prefix: "欢迎来到Sparklink查看并分享我的NFT作品：",
      },
      notes:
        "💡每个人都可以通过将链接或海报分享到作品的详细信息页面来查看和购买该作品。",
    },
  },
});

const store = useStore();
const factory = computed(
  () => store.getters["web3/userOperatorFactory"] as UserOperatorFactory
);
const infoGetter = computed(() => factory.value?.nftInformationGetter);
const route = useRoute();
const router = useRouter();
const chainId = computed(() => parseInt(route.params.chainId as string));
const chain = computed(() => chainIdToName.get(chainId.value) as string);
const isCurrentChain = computed(
  () => chainId.value === web3InfoGetter.chain.id.value
);
const nftId = computed(() => route.params.nftId as string);
const metadata = ref(null as INftInformation | null);
const account = computed(() => web3InfoGetter.account.value);
const owned = computed(() => account.value === metadata.value?.owner.value);
const toCoin = function (wei: BigInt) {
  const web3 = store.state.web3.web3 as Web3;
  return web3.utils.fromWei(wei.toString());
};
const sellingPriceInCoin = computed(() =>
  toCoin((metadata.value as INftInformation)?.sellingPrice)
);

async function resetPage() {
  metadata.value = null;
  if (nftId.value && infoGetter.value) {
    metadata.value = await infoGetter.value.get(nftId.value);
  }
}

watch([nftId, infoGetter], resetPage);
onMounted(resetPage);

watch(metadata, () => console.log(metadata.value));

const tokenAddress = computed(() => metadata.value?.paymentCurrency.value);
const tokenInquirer = web3InfoGetter.tokenInquirer;
const token = ref({ symbol: "Unknown" } as IToken);

async function setToken() {
  if (tokenAddress.value && tokenInquirer.value) {
    token.value = (await tokenInquirer.value.query(
      tokenAddress.value
    )) as IToken;
  }
}

watch([tokenAddress, tokenInquirer], setToken);
onMounted(setToken);

const baseUrl = computed(() => store.state.config.frontendBaseUrl);
const shareLink = computed(
  () => `${baseUrl.value}/#/spark/${route.params.chainId}/${route.params.nftId}`
);

function copyLink() {
  const clipboard = navigator.clipboard;
  clipboard.writeText(`${t("copyLink.prefix")}${shareLink.value}`);
  ElMessage({
    type: "success",
    message: t("copyLink.hint"),
  });
  showShareDialog.value = false;
}

const canDownload = computed(() => !!factory.value && !!metadata.value);
const eventEmitter = ref(null as DownloadEventEmitter | null);
const downloader = ref(null as IContentDownloader | null);
const encrypted = computed(() => metadata.value?.encrypted ?? false);
const downloadInProcess = ref(false);

function setDownloader(
  emitter: DownloadEventEmitter,
  _downloader: IContentDownloader
) {
  eventEmitter.value = emitter;
  downloader.value = _downloader;
  downloadInProcess.value = true;
}

function resetDownloader() {
  eventEmitter.value = null;
  downloader.value = null;
}

watch(downloadInProcess, (a1, a2) => {
  if (!downloadInProcess.value) {
    resetDownloader();
  }
});

async function clickDownloadButton() {
  const _meta = metadata.value as INftInformation;
  if (!canDownload.value) return;
  const downloader = await factory.value.getContentDownloader(
    encrypted.value,
    nftId.value
  );
  const eventEmitter = new DownloadEventEmitter();
  setDownloader(eventEmitter, downloader);
}

const showShareDialog = ref(false);
const showPoster = ref(false);
const showClaimInProgress = ref(false);

// Mint
</script>

<style lang="scss" scoped>
.art-detail {
  max-width: 1920px;
  margin: 0 auto;
  position: relative;
  padding: 142px 0 175px 0;

  .btn {
    height: 70px;
    padding: 22px 0;
    cursor: pointer;
    font-size: 24px;
    line-height: 26px;
    font-weight: 700;
    border: none;
    border-radius: 15px;
  }

  .btn.orange {
    color: white;
    background: #ef7a61;
  }

  .btn.white {
    color: #ef7a61;
    border: 2px solid #ef7a61;
    background: white;
  }

  .back-to-collection {
    width: 73px;
    position: absolute;
    top: 130px;
    left: 228px;
    cursor: pointer;
  }

  .title {
    margin: 0;
    font-size: 48px;
    font-weight: 900;
    line-height: 48px;
    text-align: center;
  }

  .current-earnings {
    margin-top: 92px;
    text-align: center;

    .note {
      margin: 0 0 17px;
      color: #777777;
      font-style: normal;
      font-weight: 700;
      font-size: 24px;
    }

    .result {
      margin-bottom: 43px;
    }

    .result p {
      display: inline-block;
      margin: 0;
      font-style: normal;
      font-weight: 800;
      font-size: 36px;
      line-height: 44px;
    }

    .result {
      .number {
        margin-right: 20px;
        color: #1e1e1e;
      }

      .unit {
        color: #ef7a61;
      }
    }

    .receive {
      margin-right: 93px;
    }

    .btns > button {
      width: 395px;
    }
  }

  .detail-container {
    max-width: 1425px;
    display: flex;
    justify-content: space-between;
    margin: 113px auto 0 auto;

    .cover {
      max-width: 525px;
      flex: 1;
      text-align: center;

      .cover-container {
        width: 100%;
        height: 525px;
        display: flex;
        align-items: center;

        & > img {
          width: 100%;
          height: 100%;
          object-fit: cover;
          border-radius: 15px;
        }
      }

      .download {
        width: 364px;
        margin-top: 73px;
      }
    }

    .info {
      max-width: 777px;
      flex: 1;
      text-align: left;

      .ntf-id {
        margin: 0;
        font-style: normal;
        font-weight: 600;
        font-size: 24px;
        line-height: 29px;
        color: #ef7a61;
      }

      .nft-detail {
        margin: 106px 0;

        p {
          margin: 0;
          font-style: normal;
          font-weight: 600;
          font-size: 20px;
          line-height: 100%;
          color: #303030;
        }

        p + p {
          margin-top: 29px;
        }
      }

      .btn {
        width: 375px;
      }

      .btn + .btn {
        margin-left: 27px;
      }

      .notes {
        margin: 27px 0 0;
        font-style: normal;
        font-weight: 500;
        font-size: 20px;
        line-height: 170%;
        color: #464646;
      }
    }
  }
}

.art-container {
  display: flex;
  margin-top: 16px;
  flex-direction: column;
  justify-content: stretch;
  width: 100%;

  &.mobile {
    align-items: stretch;

    h2,
    p {
      text-align: center;
    }

    .description {
      margin-block-start: unset;
    }
  }

  .description-area {
    border: unset;

    & > :deep(.el-card__body) {
      display: flex;
      flex-direction: column;
      row-gap: 16px;
      padding: unset;
    }

    .top {
      display: flex;
      justify-content: space-between;
      margin-bottom: -16px;

      .nftId {
        font-weight: bold;
      }
    }
  }

  .description {
    margin-top: 12px;
    margin-bottom: 12px;

    color: var(--el-color-info);
  }
}

.area + .area {
  margin-top: 16px;
}

.share-area {
  display: flex;
  justify-content: end;
}

.mobile {
  .share-area {
    flex-direction: column;
    gap: 16px;

    :deep(.el-button + .el-button) {
      margin-left: unset;
    }
  }
}

.ntf-name {
  margin: 22px 0 0;
  font-style: normal;
  font-weight: 800;
  font-size: 36px;
  line-height: 44px;
  color: #2c2f30;
}
</style>
