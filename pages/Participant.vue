<template>
  <div class="company-profile_participant">
    <SmtAlert
      v-if="beingEditedCompanyTitle"
      type="warning"
      show-icon
      :title="$t('companyProfileEdit')"
      :desc="beingEditedCompanyTitle"
      class="company-profile_participant-alert"
    />
    <CompanyProfileParticipantHeader
      :tree="tree"
      :selected-tree-item="selectedTreeItem"
      @setItemExpanded="setItemExpanded"
      @setItemChecked="setItemChecked"
      @openIpModal="openIpModal"
      @openUserModal="openUserModal"
      @openDepartmentModal="openDepartmentModal"
    />
    <CompanyProfileParticipantUsers
      v-model="userFilter"
      :users="usersWithFilter"
      :departments="departments"
      @openUserModal="openUserModal"
      @openBlockUserModal="openBlockUserModal"
    />
    <CompanyProfileParticipantIpsModal
      ref="ipModal"
      :access-policy-ips="accessPolicyIps"
      @on-reload="getCompanyStructureView"
    />
    <CompanyProfileParticipantUserModal
      ref="userModel"
      :departments="departments"
      :organizations="organizations"
      :user-roles="userRoles"
      @on-reload="getCompanyStructureView"
    />
    <CompanyProfileParticipantDepartmentModal
      ref="departmentModal"
      :organizations="organizations"
      @on-reload="getCompanyStructureView"
    />
    <CompanyProfileParticipantBlockModal
      ref="blockUserModal"
      @on-reload="getCompanyStructureView"
    />
  </div>
</template>
<script>
import breadCrumbs from '~/mixins/breadCrumbs';
import CompanyProfileParticipantHeader from '~/components/Pages/CompanyProfile/Participant/CompanyProfileParticipantHeader';
import CompanyProfileParticipantUsers from '~/components/Pages/CompanyProfile/Participant/CompanyProfileParticipantUsers';
import CompanyProfileParticipantIpsModal from '~/components/Pages/CompanyProfile/Participant/CompanyProfileParticipantIpsModal';
import CompanyProfileParticipantDepartmentModal from '~/components/Pages/CompanyProfile/Participant/CompanyProfileParticipantDepartmentModal';
import CompanyProfileParticipantBlockModal from '~/components/Pages/CompanyProfile/Participant/CompanyProfileParticipantBlockModal';
import CompanyProfileParticipantUserModal from '~/components/Pages/CompanyProfile/Participant/CompanyProfileParticipantUserModal';

export default {
  name: 'Participant',
  components: {
    CompanyProfileParticipantHeader,
    CompanyProfileParticipantUsers,
    CompanyProfileParticipantIpsModal,
    CompanyProfileParticipantDepartmentModal,
    CompanyProfileParticipantUserModal,
    CompanyProfileParticipantBlockModal,
  },
  mixins: [
    breadCrumbs,
  ],
  nuxtI18n: {
    paths: {
      uk: '/prozorro/company-profile/participant/:organizationId?',
      ru: '/prozorro/company-profile/participant/:organizationId?',
      en: '/prozorro/company-profile/participant/:organizationId?',
    },
  },
  async asyncData({ $axios, error, route }) {
    try {
      const organizationId = route.params.organizationId || '';
      const COMPANY_STRUCTURE_URL = `/ProfileParticipants/companyStructure/${organizationId}`;
      const { data: model } = await $axios.get(COMPANY_STRUCTURE_URL);
      return { model, COMPANY_STRUCTURE_URL };
    } catch (e) {
      error({ statusCode: e.response ? e.response.status : 500 });
    }
  },
  data() {
    return {
      accessPolicyIps: [],
      organizations: [],
      departments: [],
      userRoles: [],
      users: [],
      beingEditedCompanyTitle: '',
      tree: [],
      linearTree: {},
      selectedTreeItemsId: 0,
      userFilter: {
        name: '',
        email: '',
        status: true,
        department: '',
      },
    };
  },
  head() {
    return {
      title: this.$t('companyProfile'),
    };
  },
  computed: {
    breadCrumbs() {
      return [
        { Title: this.$t('mainPage'), Link: '/' },
        { Title: this.$t('companyProfile') },
      ];
    },
    selectedTreeItem() {
      return this.linearTree[this.selectedTreeItemsId] || {};
    },
    usersWithFilter() {
      if (this.users.length) {
        let newDataUsers = this.users.filter(user => (
          (user.name || '').toLowerCase().includes(this.userFilter.name.toLowerCase()) &&
          (user.email || '').toLowerCase().includes(this.userFilter.email.toLowerCase()) &&
          (this.userFilter.status === null ? true : this.userFilter.status === user.isActive) &&
          (user.departmentTitle || '').toLowerCase().includes(this.userFilter.department.toLowerCase())
        ));
        newDataUsers = newDataUsers.filter((user) => {
          let result;
          if (this.selectedTreeItem.id === 0) {
            result = true;
          } else if (this.selectedTreeItem.structureType === 1) {
            result = this.selectedTreeItem.id === String(user.organizationId);
          } else if (this.selectedTreeItem.structureType === 2) {
            result = this.selectedTreeItem.id === String(user.departmentId);
          }
          return result;
        });
        return newDataUsers;
      }
      return [];
    },
  },
  created() {
    this.setModel(this.model);
  },
  methods: {
    getCompanyStructureView() {
      this.$spinner.start();
      this.$axios.get(this.COMPANY_STRUCTURE_URL)
        .then((resp) => {
          this.setModel(resp.data);
        })
        .catch(() => {
          this.$Notice.error({
            title: this.$t('error'),
          });
        })
        .finally(this.$spinner.finish);
    },
    setModel(model) {
      this.accessPolicyIps = model.accessPolicyIps;
      this.departments = model.departments;
      this.organizations = model.organizations;
      this.userRoles = model.userRoles;
      this.beingEditedCompanyTitle = model.beingEditedCompanyTitle;
      this.users = model.users;
      this.createTree(model.companyTree);
      this.createLinearTree();
      this.setAllUsersDepartment();
    },
    createTree(data) {
      this.tree = [{
        id: 0,
        text: this.$t('allCompany'),
        state: { expanded: true, checked: true },
        children: data,
        structureType: 1,
      }];
    },
    createLinearTree() {
      const linearTree = {};
      let children = this.tree;
      while (children.length > 0) {
        children = children.reduce((total, item) => {
          linearTree[item.id] = item;
          if (item.children) {
            item.children.forEach(child => total.push(child));
          }
          return total;
        }, []);
      }
      this.linearTree = linearTree;
    },
    setAllUsersDepartment() {
      for (const user of this.users) {
        const userDepartment = user.departmentId ? this.departments.find(d => d.id === user.departmentId) : null;
        if (userDepartment) {
          user.departmentTitle = userDepartment.title;
        } else {
          user.departmentTitle = '';
        }
      }
    },
    setItemExpanded(id) {
      this.linearTree[id].state.expanded = !this.linearTree[id].state.expanded;
    },
    setItemChecked(id) {
      if (!this.linearTree[id].state.checked) {
        for (const key in this.linearTree) {
          if (this.linearTree[key].state.checked) {
            this.linearTree[key].state.checked = false;
          }
        }
        this.linearTree[id].state.checked = true;
        this.selectedTreeItemsId = id;
      }
    },
    openIpModal() {
      this.$refs.ipModal.openModal();
    },
    openUserModal(user, department) {
      this.$refs.userModel.openModal(user, department);
    },
    openDepartmentModal(department) {
      this.$refs.departmentModal.openModal(department);
    },
    openBlockUserModal(user, state) {
      this.$refs.blockUserModal.openModal(user, state);
    },
  },
  i18n: {
    messages: {
      uk: {
        mainPage: 'SmartTender',
        companyProfile: 'Структура підприємства',
        allCompany: 'Вся компанія',
        companyProfileEdit: 'Коригування профілю компанії',
      },
      ru: {
        mainPage: 'SmartTender',
        companyProfile: 'Структура предприятия',
        allCompany: 'Вся компания',
        companyProfileEdit: 'Корректировка профиля компании',
      },
      en: {
        mainPage: 'SmartTender',
        companyProfile: 'The structure of the enterprise',
        allCompany: 'All company',
        companyProfileEdit: 'Adjusting the company profile',
      },
    },
  },
};
</script>
<style lang="sass">
.company-profile_participant
  background: white
  padding: 34px 36px
  .company-profile_participant-alert
    margin-bottom: 30px
</style>
