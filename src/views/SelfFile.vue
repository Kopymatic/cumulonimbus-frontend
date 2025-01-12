<template>
  <h1>File Details</h1>
  <h2>See the specifics of a file.</h2>
  <div class="quick-action-buttons-container">
    <button @click="$router.replace('/dashboard/files/')" title="Go back!"
      >Back</button
    >
    <button @click="$refs.deleteFileModal.show()" title="Delete this file!"
      >Delete</button
    >
    <button @click="shareFile" title="Share file!">Share</button>
  </div>
  <div v-if="loaded" class="content-box-group-container">
    <ContentBox
      title="Information"
      class="file-info-box"
      span
      lazy-load
      new-tab
      :to="`https://${hostname}/${file.filename}`"
      :src="`https://previews.${hostname}/${file.filename}`"
    >
      <p
        >Filename: <code>{{ file.filename }}</code></p
      >
      <p
        >Uploaded at:
        <code>{{
          $parent.$parent.toDateString(new Date(file.createdAt))
        }}</code></p
      >
      <p
        >Size: <code>{{ hFileSize }}</code></p
      >
    </ContentBox>
  </div>
  <Loading v-else />
  <Modal title="Delete file" ref="deleteFileModal" cancelable>
    <template v-slot:default>
      <p
        >Are you sure you want to delete <code>{{ file.filename }}</code
        >? Unless you have it saved somewhere you will never see it again!</p
      >
    </template>
    <template v-slot:buttons>
      <button @click="deleteFile" title="Bye bye file!">I'm sure</button>
      <button
        @click="$refs.deleteFileModal.hide()"
        title="That's what I thought."
        >On second thought...</button
      >
    </template>
  </Modal>
</template>

<script lang="ts">
  import { Options, Vue } from 'vue-class-component';
  import ContentBox from '@/components/ContentBox.vue';
  import { Client, Cumulonimbus } from '../../../cumulonimbus-wrapper';
  import Modal from '@/components/Modal.vue';
  import Loading from '@/components/Loading.vue';
  import App from '@/App.vue';

  function formatBytes(bytes: number, decimals = 2) {
    if (bytes === 0) return '0 Bytes';

    const k = 1024;
    const dm = decimals < 0 ? 0 : decimals;
    const sizes = ['Bytes', 'KB', 'MB', 'GB', 'TB', 'PB', 'EB', 'ZB', 'YB'];

    const i = Math.floor(Math.log(bytes) / Math.log(k));

    return parseFloat((bytes / Math.pow(k, i)).toFixed(dm)) + ' ' + sizes[i];
  }

  @Options({
    components: { ContentBox, Modal, Loading },
    data() {
      return {
        file: null,
        hostname: null,
        loaded: false,
        hFileSize: null,
        canShare: false
      };
    },
    title: 'File Details'
  })
  export default class VueComponent extends Vue {
    declare $data: {
      file: Cumulonimbus.Data.File | null;
      hostname: string | null;
      loaded: boolean;
      hFileSize: string | null;
      canShare: boolean;
    };
    declare $refs: {
      deleteFileModal: Modal;
    };
    async mounted() {
      this.$data.canShare = !!navigator.share;
      if (!navigator.onLine) {
        (this.$parent?.$parent as App).temporaryToast(
          "Looks like you're offline, I'm pretty useless offline.",
          5000
        );
        return;
      }
      this.$data.hostname = window.location.hostname;

      try {
        const urlSearchParams = new URLSearchParams(window.location.search);
        if (!this.$store.state.user) {
          let u = await this.$store.state.client.getSelfUser();
          this.$store.commit('setUser', u);
        }
        if (!urlSearchParams.has('id')) this.$router.push('/dashboard/files/');
        this.$data.file = await this.$store.state.client.getSelfFileByID(
          urlSearchParams.get('id')
        );
        this.$data.hFileSize = formatBytes(this.$data.file?.size as number);
        this.$data.loaded = true;
      } catch (error) {
        if (error instanceof Cumulonimbus.ResponseError) {
          switch (error.code) {
            case 'RATELIMITED_ERROR':
              (this.$parent?.$parent as App).ratelimitToast(
                error.ratelimit.resetsAt
              );
              break;
            case 'INVALID_SESSION_ERROR':
              (this.$parent?.$parent as App).redirectIfNotLoggedIn(
                window.location.pathname
              );
              break;
            case 'INVALID_FILE_ERROR':
              this.$router.push('/dashboard/files/');
              break;
            case 'INTERNAL_ERROR':
              (this.$parent?.$parent as App).temporaryToast(
                'The server did something weird, lets try again later.',
                5000
              );
              break;
            default:
              (this.$parent?.$parent as App).temporaryToast(
                'I did something weird, lets try again later.',
                5000
              );
              console.error(error);
          }
        } else {
          (this.$parent?.$parent as App).temporaryToast(
            'I did something weird, lets try again later.',
            5000
          );
          console.error(error);
        }
      }
    }

    async deleteFile() {
      try {
        let res = await (this.$store.state.client as Client).deleteSelfFileByID(
          this.$data.file?.filename as string
        );
        console.log(res);
        (this.$parent?.$parent as App).temporaryToast('Done!', 2000);
        this.$refs.deleteFileModal.hide();
        this.$router.push('/dashboard/files/');
      } catch (error) {
        if (error instanceof Cumulonimbus.ResponseError) {
          switch (error.code) {
            case 'INVALID_SESSION_ERROR':
              (this.$parent?.$parent as App).redirectIfNotLoggedIn(
                window.location.pathname
              );
              break;
            case 'INVALID_FILE_ERROR':
              this.$router.push('/dashboard/files/');
              (this.$parent?.$parent as App).temporaryToast(
                'Sometime between when you got here and now this file has been deleted.',
                5000
              );
              break;
            case 'BANNED_ERROR':
              (this.$parent?.$parent as App).temporaryToast(
                "Uh oh, looks like you've been banned from Cumulonimbus, sorry for the inconvenience.",
                5000
              );
              (this.$parent?.$parent as App).redirectIfNotLoggedIn(
                window.location.pathname
              );
              break;
            case 'INTERNAL_ERROR':
              (this.$parent?.$parent as App).temporaryToast(
                'The server did something weird, lets try again later.',
                5000
              );
              break;
            default:
              (this.$parent?.$parent as App).temporaryToast(
                'I did something weird, lets try again later.',
                5000
              );
              console.error(error);
          }
        }
      }
    }

    async shareFile() {
      if (this.$data.canShare) {
        try {
          await navigator.share({
            title: this.$data.file?.filename,
            text: 'Check this out on Cumulonimbus!',
            url: `https://${
              this.$store.state.user?.subdomain
                ? `${this.$store.state.user.subdomain}.`
                : ''
            }${this.$store.state.user?.domain}/${this.$data.file?.filename}`
          });
        } catch (error) {
          if (error instanceof DOMException) return;
          (this.$parent?.$parent as App).temporaryToast(
            'I did something weird, lets try again later.',
            5000
          );
          console.error(error);
        }
      } else {
        this.copyToClipboard();
      }
    }

    copyToClipboard() {
      navigator.clipboard
        .writeText(
          `https://${
            this.$store.state.user?.subdomain
              ? `${this.$store.state.user.subdomain}.`
              : ''
          }${this.$store.state.user?.domain}/${this.$data.file?.filename}`
        )
        .then(
          () => {
            (this.$parent?.$parent as App).temporaryToast(
              'Copied to your clipboard!'
            );
          },
          err => {
            (this.$parent?.$parent as App).temporaryToast(
              "I still wasn't able to do it, it might be a browser permission issue. Let us know about it in the discord please!",
              10000
            );
            console.error(err);
          }
        );
    }
  }
</script>

<style>
  .file-info-box {
    width: 100%;
  }
</style>
