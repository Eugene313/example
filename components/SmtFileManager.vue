<template>
  <div :class="classes">
    <SmtFileManagerHeader
      :max-file-size="maxFileSize"
      @on-choose-files="chooseFiles"
    />
    <SmtFileManagerActions
      :checked-files="checkedFiles"
      :files-count="filesCount"
      :sign-files="signFiles"
      @on-sign-files-start="signFilesStart"
      @on-delete-files="deleteCheckedFiles"
      @on-cancel-selection="checkFilesHandler"
    />
    <SmtFileManagerCols
      :files="files"
      :files-count="filesCount"
      :document-types="documentTypes"
      :confidence="confidence"
      @on-check-files="checkFilesHandler"
    />
    <div class="smt-file-manager_items">
      <SmtFileManagerItem
        v-for="(file, fileIndex) in files"
        :key="file.fileName"
        v-model="files[fileIndex]"
        :file-index="fileIndex"
        :document-types="documentTypes"
        :confidence="confidence"
        :statuses="statuses"
        @on-reload="reloadFile"
        @on-delete="deleteFile"
      />
    </div>
    <SmtFileManagerDocumentTypesModal
      ref="smtFileManagerDocumentTypesModal"
      v-model="documentTypeId"
      :document-types="documentTypes"
      @on-choose-files="openUploadFileModal"
    />
    <SmtFileManagerUploadLocalFileModal
      ref="smtFileManagerUploadLocalFileModal"
      :files="files"
      :multiple="multiple"
      :forbidden-file-types="forbiddenFileTypes"
      :max-file-size="maxFileSize"
      @on-upload="addFiles"
    />
    <SmtFileManagerUploadCloudFileModal
      ref="smtFileManagerUploadCloudFileModal"
      @on-upload="addCloudFiles"
    />
  </div>
</template>
<script>
import SmtFileManagerHeader from '~/components/Common/FileManager/SmtFileManagerHeader';
import SmtFileManagerActions from '~/components/Common/FileManager/SmtFileManagerActions';
import SmtFileManagerCols from '~/components/Common/FileManager/SmtFileManagerCols';
import SmtFileManagerItem from '~/components/Common/FileManager/SmtFileManagerItem';
import SmtFileManagerDocumentTypesModal from '~/components/Common/FileManager/SmtFileManagerDocumentTypesModal';
import SmtFileManagerUploadLocalFileModal from '@/components/Common/FileManager/SmtFileManagerUploadLocalFileModal';
import SmtFileManagerUploadCloudFileModal from '@/components/Common/FileManager/SmtFileManagerUploadCloudFileModal';

export default {
  name: 'SmtFileManager',
  components: {
    SmtFileManagerHeader,
    SmtFileManagerActions,
    SmtFileManagerCols,
    SmtFileManagerItem,
    SmtFileManagerDocumentTypesModal,
    SmtFileManagerUploadLocalFileModal,
    SmtFileManagerUploadCloudFileModal,
  },
  props: {
    value: {
      type: Array,
      default: () => [],
    },
    size: {
      type: String,
      default: 'large',
      validator(value) {
        return ['large', 'medium', 'small'].includes(value);
      },
    },
    multiple: {
      type: Boolean,
      default: true,
    },
    uploadCount: {
      type: Number,
      default: 2,
    },
    documentTypes: {
      type: Array,
      default: null,
    },
    maxFileSize: {
      type: Number,
      default: 50,
    },
    confidence: {
      type: Boolean,
      default: false,
    },
    descriptionDecision: {
      type: Boolean,
      default: false,
    },
    position: {
      type: String,
      default: 'top-left',
    },
    dateCreate: {
      type: Boolean,
      default: true,
    },
    signFiles: {
      type: Boolean,
      default: true,
    },
    checked: {
      type: Boolean,
      default: false,
    },
  },
  data() {
    return {
      currentSize: 'large',
      documentTypeId: null,
      forbiddenFileTypes: [
        'exe',
        'com',
        'bat',
        'msi',
        'dll',
        'cmd',
        'aspx',
      ],
      mode: null,
      statuses: {
        upload: {
          title: this.$t('uploaded'),
          color: '#6AB42D',
          transition: 0,
        },
        pending: {
          title: this.$t('documentPending'),
          color: 'transparent',
          transition: 0,
        },
        uploading: {
          title: this.$t('loading'),
          color: '#6AB42D',
          transition: 0.3,
        },
        errorUpload: {
          title: this.$t('uploadError'),
          color: '#A7676F',
          transition: 0,
          background: '#f5caca',
        },
      },
    };
  },
  computed: {
    files: {
      get() {
        return this.value;
      },
      set(v) {
        this.$emit('input', v);
      },
    },
    isLocalMode() {
      return this.mode === 'local';
    },
    isCloudMode() {
      return this.mode === 'cloud';
    },
    filesCount() {
      return this.files ? this.files.length : 0;
    },
    isDocumentTypesAvailable() {
      return this.documentTypes && !!this.documentTypes.length;
    },
    checkedFiles() {
      return this.files.filter(file => file.checked);
    },
    pendingFilesCount() {
      return this.files.filter(f => f.pending).length;
    },
    uploadingFilesCount() {
      return this.files.filter(f => f.uploading).length;
    },
    classes() {
      return [
        'smt-file-manager',
        { 'smt-file-manager_large': this.currentSize === 'large' },
        { 'smt-file-manager_medium': this.currentSize === 'medium' },
        { 'smt-file-manager_small': this.currentSize === 'small' },
      ];
    },
  },
  watch: {
    pendingFilesCount() {
      this.checkPendingFiles();
    },
  },
  methods: {
    setMode(mode) {
      this.mode = mode;
    },
    openDocumentTypesModal() {
      this.$refs.smtFileManagerDocumentTypesModal.openModal();
    },
    openUploadLocalFileModal() {
      this.$refs.smtFileManagerUploadLocalFileModal.openModal();
    },
    openUploadCloudFileModal() {
      this.$refs.smtFileManagerUploadCloudFileModal.openModal();
    },
    openUploadFileModal() {
      if (this.isLocalMode) {
        this.openUploadLocalFileModal();
      } else if (this.isCloudMode) {
        this.openUploadCloudFileModal();
      }
    },
    chooseFiles(mode) {
      this.setMode(mode);
      if (this.isDocumentTypesAvailable) {
        this.openDocumentTypesModal();
      } else {
        this.openUploadFileModal();
      }
    },
    createFile(file) {
      return {
        fileName: file.name,
        localFile: file,
        uploaded: false,
        uniqueId: null,
        dateCreated: this.$moment().toDate(),
        downloadUrl: null,
        md5Hash: null,
        previousVersions: [],
        documentTypeId: this.isDocumentTypesAvailable ? this.documentTypeId : null,
        isConfident: false,
        confidentialityRationale: '',
        isDescriptionDecision: false,
      };
    },
    createFiles(files) {
      return [...files].map(file => this.createFile(file));
    },
    createCloudFile(file) {
      return {
        ...file,
        documentTypeId: this.isDocumentTypesAvailable ? this.documentTypeId : null,
      };
    },
    createCloudFiles(files) {
      return [...files].map(file => this.createCloudFile(file));
    },
    setFiles(files) {
      files.forEach((file) => {
        this.files.push(file);
      });
    },
    addFiles(files) {
      const filesData = this.createFiles(files);
      this.setFiles(filesData);
    },
    addCloudFiles(files) {
      const filesData = this.createCloudFiles(files);
      this.setFiles(filesData);
    },
    checkPendingFiles() {
      if (this.pendingFilesCount && this.uploadingFilesCount <= this.uploadCount) {
        const files = this.files.filter(file => file.pending && !file.uploading).slice(0, this.uploadCount);
        this.uploadFiles(files);
      }
    },
    async uploadFile(file) {
      try {
        const formData = new FormData();
        formData.append('file', file.localFile);
        file.uploading = true;
        const { data } = await this.$axios.post('/Documents', formData, {
          onUploadProgress(progressEvent) {
            file.progress = Math.round((progressEvent.loaded * 100) / progressEvent.total);
          },
        });
        file.uploaded = true;
        file.downloadUrl = data.downloadUrl;
        file.md5Hash = data.md5Hash;
        file.uniqueId = data.uniqueId;
        this.$emit('on-upload', file);
      } catch (e) {
        file.progress = 100;
        file.error = true;
        if (e.response && e.response.data.errors) {
          file.errorText = e.response.data.errors[0].message;
        } else {
          file.errorText = '';
        }
      } finally {
        file.pending = false;
        file.uploading = false;
      }
    },
    reloadFile(file) {
      file.error = false;
      file.errorText = '';
      file.progress = 0;
      this.$nextTick(() => {
        file.pending = true;
      });
    },
    uploadFiles(files) {
      for (const file of files) {
        this.uploadFile(file);
      }
    },
    deleteFile(file) {
      const fileIndex = this.files.findIndex(f => f.fileName === file.fileName);
      this.files.splice(fileIndex, 1);
    },
    deleteCheckedFiles() {
      this.checkedFiles.forEach((file) => {
        this.deleteFile(file);
      });
    },
    checkFiles() {
      this.files.forEach((file) => {
        file.checked = true;
      });
    },
    unCheckFiles() {
      this.files.forEach((file) => {
        file.checked = false;
      });
    },
    checkFilesHandler(value) {
      if (value) {
        this.checkFiles();
      } else {
        this.unCheckFiles();
      }
    },
    signFilesStart() {
      this.$Signature.sign({
        mode: 'signFile',
        signData: this.checkedFiles,
        onSuccess: this.signFilesFinish,
        onClose: this.unCheckFiles,
      });
    },
    async getFileSignature(param) {
      try {
        this.$spinner.start();
        const response = await this.$axios.post('/Signature/fileSignature', {
          filename: param.file.fileName,
          signature: param.sign,
        });
        return { file: param.file, signFile: response.data };
      } catch (e) {
        this.$Notice.error({
          title: this.$t('error'),
          desc: this.$t('errorCreateFileSign'),
        });
      } finally {
        this.$spinner.finish();
      }
    },
    async signFilesFinish({ sign }) {
      try {
        const filesSignature = await Promise.all(sign.map(param => this.getFileSignature(param)));
        filesSignature.forEach((fileSignature) => {
          const file = fileSignature.file;
          const fileIndex = this.files.findIndex(f => f.downloadUrl === file.downloadUrl);
          const signFile = fileSignature.signFile;
          const newFileSignature = this.createFile(file);
          newFileSignature.fileName = `${file.fileName}.p7s`;
          newFileSignature.downloadUrl = signFile.downloadUrl;
          newFileSignature.md5Hash = signFile.md5Hash;
          newFileSignature.uniqueId = signFile.uniqueId;
          this.files.splice(fileIndex + 1, 0, newFileSignature);
        });
        this.unCheckFiles();
      } catch (error) {
        this.$Notice.error({
          title: this.$t('error'),
          error,
        });
      }
    },
  },
  i18n: {
    messages: {
      uk: {
        error: 'Помилка',
        errorCreateFileSign: 'Не вдалося створити файл підпису',
        uploaded: 'Завантажено',
        documentPending: 'Очікує завантаження',
        loading: 'Завантажується',
        uploadError: 'Не завантажено',
      },
      ru: {
        error: 'Ошибка',
        errorCreateFileSign: 'Не удалось создать файл подписи',
        uploaded: 'Загружен',
        documentPending: 'Ожидает загрузки',
        loading: 'Загружается',
        uploadError: 'Не загружен',
      },
      en: {
        error: 'Error',
        errorCreateFileSign: 'Failed to create signature file',
        uploaded: 'Uploaded',
        documentPending: 'Awaiting upload',
        loading: 'Loading',
        uploadError: 'Not uploaded',
      },
    },
  },
};
</script>
<style lang="sass">
.smt-file-manager_header
  display: flex
  flex-wrap: wrap
  padding: 0 0 20px 0
  .smt-file-manager_header-button
    min-width: 162px
    height: 32px
    margin-bottom: 10px
    margin-right: 10px
  .smt-file-manager_header-info
    width: 100%
    font-size: 12px
    line-height: 14px
    color: #929292
.smt-file-manager_actions
  position: relative
  display: flex
  justify-content: space-between
  align-items: center
  background: #EDF2F7
  padding: 0 6px
  height: 50px
  .smt-file-manager_actions-buttons
    display: flex
    align-items: center
  .smt-file-manager_actions-button
    margin: 0 0 0 24px
  .smt-file-manager_actions-button-delete
    color: #A7676F
    svg
      fill: #A7676F
  .smt-file-manager_actions-button-cancel
    box-shadow: 0 1px 4px #B9CFE8
@mixin col
  display: flex
  align-items: center
  height: 46px
  padding: 6px 6px 6px 0
.smt-file-manager_cols
  position: relative
  display: flex
  padding: 0 20px
  font-size: 12px
  .smt-file-manager_cols-check
    @include col
    width: 50px
  .smt-file-manager_cols-number
    @include col
    width: 30px
  .smt-file-manager_cols-title
    @include col
    flex: 11
  .smt-file-manager_cols-date
    @include col
    flex: 4
  .smt-file-manager_cols-type
    @include col
    flex: 8
  .smt-file-manager_cols-confidence
    @include col
    flex: 4
  .smt-file-manager_cols-actions
    @include col
    flex: 3
@mixin item-col
  display: flex
  align-items: center
  min-height: 52px
  padding: 6px 6px 6px 0
.smt-file-manager_item
  position: relative
  border-bottom: 1px solid #eaecee
  &:last-child
    border-bottom: none
  .smt-file-manager_item-cols
    position: relative
    display: flex
    padding: 0 20px
    font-size: 14px
  .smt-file-manager_item-cols-check
    @include item-col
    width: 50px
  .smt-file-manager_item-cols-number
    @include item-col
    width: 30px
  .smt-file-manager_item-cols-title
    @include item-col
    flex: 11
    display: flex
    align-items: center
    flex-wrap: wrap
    word-wrap: break-word
  .smt-file-manager_item-cols-date
    @include item-col
    flex: 4
  .smt-file-manager_item-cols-type
    @include item-col
    flex: 8
  .smt-file-manager_item-cols-confidence
    @include item-col
    flex: 4
  .smt-file-manager_item-cols-actions
    @include item-col
    flex: 3
    display: flex
    justify-content: flex-end
  .smt-file-manager_item-actions-button
    width: 30px
    height: 30px
    background: none
    border: none
    box-shadow: none
    cursor: pointer
    border-radius: 50%
    display: flex
    justify-content: center
    align-items: center
    transition: 0.3s
    svg
      height: 20px
  .smt-file-manager_item-confident
    padding: 0 20px 0 80px
    height: 60px
    display: flex
    align-items: center
  .smt-file-manager_item-message
    height: 28px
    display: flex
    align-items: center
    font-size: 12px
    background: #c2c2c2
    padding: 0 11%
  .smt-file-manager_item-progress
    position: absolute
    bottom: 0
    right: 15px
    left: 15px
    height: 3px
    border-radius: 3px
    overflow: hidden
    background: #F4F6F5
  .smt-file-manager_item-delete-pop-tip
    background: #EDF2F7
    .wrap
      padding: 0 20px 0 20px
.smt-file-manager_item-previous
  background: #F2F2F2
  &:first-child
    box-shadow: inset 0 10px 10px -16px
  .smt-file-manager_item-cols-title
    text-decoration: line-through
.smt-file-manager_item-previous-button
  display: flex
  align-items: center
  font-size: 12px
  text-transform: uppercase
  color: #014A90
  cursor: pointer
  width: 100%
  margin: 12px 0 0 0
  user-select: none
  svg
    fill: #014A90
    height: 18px
.smt-file-manager_item-error
  .smt-file-manager_item-cols-number
    color: #A7676F
  .smt-file-manager_item-cols-title
    color: #A7676F
    svg
      fill: #A7676F
.smt-file-manager_icon
  margin-right: 8px
  font-size: 20px
</style>
