<template>
  <div>
    <b-loading is-full-page v-model="isLoading" :can-cancel="true"></b-loading>
    <div class="box">
      <p class="title is-size-3">
        {{ $t('context') }}
      </p>
      <p class="subtitle is-size-7">
        {{ $t('using') }} {{ version }}
      </p>
      <div>
        {{ $t('computed id') }}: <b>{{ rmrkId }}</b>
      </div>
      <AccountSelect :label="$i18n.t('Account')" v-model="accountId" />
      <b-field grouped :label="$i18n.t('Name')">
        <b-input v-model="rmrkMint.name" expanded></b-input>
        <Tooltip :label="$i18n.t('Owner address of minted art')" />
      </b-field>
      <b-field :label="$i18n.t('Maximum NFTs in collection')">
        <b-numberinput
          v-model="rmrkMint.max"
          placeholder="1 is minumum"
          :min="1"
        ></b-numberinput>
      </b-field>
      <b-field grouped :label="$i18n.t('Symbol')">
        <b-input v-model="rmrkMint.symbol" expanded></b-input>
        <Tooltip :label="$i18n.t('Symbol you want to trade it under')" />
      </b-field>
      <p class="title">
        {{ $t('content')}}
      </p>
      <b-switch v-model="uploadMode" 
        passive-type="is-dark"
        :rounded="false">
        {{ uploadMode ? 'Upload through KodaDot' : 'IPFS hash of your content' }}
      </b-switch>
      <template v-if="uploadMode">
        <b-field :label="$i18n.t('Description')">
          <b-input
            v-model="meta.description"
            maxlength="200"
            type="textarea"
          ></b-input>
        </b-field>
        <MetadataUpload v-model="image" />
        <b-field :label="$i18n.t('Multimedia data')">
          <b-input v-model="meta.image_data"></b-input>
        </b-field>
      </template>

      <b-field v-else :label="$i18n.t('Metadata IPFS Hash')">
        <b-input v-model="rmrkMint.metadata"></b-input>
      </b-field>
      <PasswordInput v-model="password" :account="accountId" />   
      <b-button
        type="is-primary"
        icon-left="paper-plane"
        @click="submit"
        :disabled="disabled"
        :loading="isLoading"
        outlined
      >
        {{ $t('create collection') }}
      </b-button>
    </div>
  </div>
</template>

<script lang="ts" >
import { Component, Prop, Vue, Mixins } from 'vue-property-decorator';
import { RmrkMint } from '../types';
import { emptyObject } from '@/utils/empty';
import AccountSelect from '@/components/shared/AccountSelect.vue';
import Tooltip from '@/components/shared/Tooltip.vue';
import MetadataUpload from './MetadataUpload.vue';
import Connector from '@vue-polkadot/vue-api';
import exec, { execResultValue, ExecResult } from '@/utils/transactionExecutor';
import { notificationTypes, showNotification } from '@/utils/notification';
import PasswordInput from '@/components/shared/PasswordInput.vue';
import SubscribeMixin from '@/utils/mixins/subscribeMixin';
import RmrkVersionMixin from '@/utils/mixins/rmrkVersionMixin';

import { getInstance, RmrkType } from '../service/RmrkService';
import { Collection, CollectionMetadata } from '../service/scheme';
import { pinFile, pinJson, unSanitizeIpfsUrl } from '@/pinata';
import { decodeAddress } from '@polkadot/keyring';
import { u8aToHex } from '@polkadot/util';
import { generateId } from '@/components/rmrk/service/Consolidator'


const components = {
  AccountSelect,
  MetadataUpload,
  PasswordInput,
  Tooltip,
};

@Component({ components })
export default class CreateCollection extends Mixins(SubscribeMixin, RmrkVersionMixin) {
  private rmrkMint: Collection = emptyObject<Collection>();
  private meta: CollectionMetadata = emptyObject<CollectionMetadata>();
  private accountId: string = '';
  private uploadMode: boolean = true;
  private image: Blob | null = null;
  private isLoading: boolean = false;
  private password: string = '';

  get rmrkId(): string {
    return generateId(this.accountId, this.rmrkMint?.symbol || '')
  }

  get accountIdToPubKey() {
    return (this.accountId && u8aToHex(decodeAddress(this.accountId))) || '';
  }

  private generateId(pubkey: string): string {
    return (
      pubkey?.substr(2, 10) +
      pubkey?.substring(pubkey.length - 8) +
      '-' +
      (this.rmrkMint?.symbol || '')
    ).toUpperCase();
  }

  get disabled(): boolean {
    const { name, symbol, max } = this.rmrkMint;
    return !(name && symbol && max && this.accountId && this.image);
  }

  public constructRmrkMint(): Collection {
    const mint: Collection = {
      ...this.rmrkMint,
      symbol: this.rmrkMint.symbol.toUpperCase(),
      version: this.version,
      issuer: this.accountId,
      metadata: unSanitizeIpfsUrl(this.rmrkMint?.metadata),
      id: this.rmrkId
    };

    return mint;
  }

  public async constructMeta() {
    if (!this.image) {
      throw new ReferenceError('No file found!');
    }

    this.meta = {
      ...this.meta,
      attributes: [],
      external_url: `https://rmrk.app/registry/${this.rmrkId}`
    };

    // TODO: upload image to IPFS
    const imageHash = await pinFile(this.image);
    this.meta.image = unSanitizeIpfsUrl(imageHash);
    // TODO: upload meta to IPFS
    const metaHash = await pinJson(this.meta);

    return unSanitizeIpfsUrl(metaHash);
  }

  private async submit() {
    this.isLoading = true;
    const { api } = Connector.getInstance();
    const rmrkService = getInstance();
    const mint = this.constructRmrkMint();
    if (!this.rmrkMint.metadata) {
      const meta = await this.constructMeta();
      mint.metadata = meta;
    }

    const mintString = `RMRK::MINT::${this.version}::${encodeURIComponent(
      JSON.stringify(mint)
    )}`;
    try {
      showNotification(mintString)
      console.log('submit', mintString);
      const tx = await exec(this.accountId, this.password, api.tx.system.remark, [
        mintString
      ], async (result) => {
        console.log(`Current status is`, result);
        if (result.status.isFinalized) {
          console.log(`finalized status is`, result);
          console.log(`Transaction finalized at blockHash ${result.status.asFinalized}`);
          execResultValue(tx)
          const header = await api.rpc.chain.getHeader(result.status.asFinalized);
          const persisted = await rmrkService?.resolve(mintString, this.accountId, header.number.toString());
          console.log('SAVED', persisted?._id);
          showNotification(`[TEXTILE] ${persisted?._id}`, notificationTypes.success)
          this.isLoading = false;
        }
      });
      console.warn('TX IN', tx);
      showNotification(`[CHAIN] Waiting to finalize block and save to TEXTILE`)
      
    } catch (e) {
      showNotification(`[ERR] ${e}`, notificationTypes.danger)
      console.error(e);
      this.isLoading = false;
    }

    
  }

  private upload(data: File) {
    console.log('upload', data.name);
    this.image = data;
  }
}
</script>
