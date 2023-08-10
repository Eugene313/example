<template>
  <div class="contracts">
    <ContractsSearchHeader
      id="contracts-header"
      v-model="filter.Phrase"
      :total-count="totalCount"
      :total-tags="tags.length"
      @on-change="setTags"
    />
    <ContractsSearchTabs
      :contracts-favorite-count="contractsFavoriteCount"
      :filter="filter"
      @on-change-tab="changeTab"
    />
    <div class="contracts-container">
      <SmtForm
        ref="filterForm"
        :model="filter"
        :rules="filterRules"
      >
        <ContractsSearchFilter
          v-model="filter"
          :default-filter="defaultFilter"
          :selected-organizers="selectedOrganizers"
          :selected-participants="selectedParticipants"
          @on-change="setTags"
        />
      </SmtForm>
      <div class="contracts-container_block">
        <ContractsSearchActions
          v-model="filter"
          :total-count="totalCount"
          @on-change="changePageSize"
        />
        <SmtTags
          v-model="tags"
          :start-count="10"
          :step="20"
          :max-tags="30"
          data-qa="tag-holder"
          @on-delete-tag="deleteFilter"
          @clear-all="clearFilterAll"
        />
        <template v-if="contracts">
          <ContractsSearchCard
            v-for="(contract, contractIndex) in contracts"
            :key="contract.number + contractIndex"
            v-model="contracts[contractIndex]"
            class="contract-card"
            @on-change-participants="searchByParticipants"
            @on-change-organizers="searchByOrganizers"
            @on-change-favorite-count="changeFavoriteCount"
          />
        </template>
        <div
          v-if="contracts && contracts.length === 0"
          :key="'nothing-found'"
          class="contracts-nothing_found"
        >
          {{ $t('nothingFound') }}
        </div>
        <SmtButton
          v-if="loadMoreCondition"
          class="contracts-button_more"
          icon="rotate-left"
          outlined
          @click="loadContracts"
        >
          {{ `${$t('loadMore', { count: loadMoreCount})}` }}
        </SmtButton>
        <SmtPagination
          v-if="paginationCondition"
          :total-count="totalCount"
          :current="filter.Page"
          :page-size="filter.CountOnPage"
          :item-count="contracts.length"
          class="block_pagination"
          @on-change="changePage"
        />
      </div>
    </div>
  </div>
</template>
<script>
import SmtForm from '@smarttender-dev/smart-ui-kit/smt-form';
import CloneDeep from 'lodash/cloneDeep';
import { mapState, mapActions } from 'vuex';
import breadCrumbs from '~/mixins/breadCrumbs';
import ContractsSearchHeader from '~/components/Pages/Contracts/ContractsSearch/ContractsSearchHeader';
import ContractsSearchFilter from '~/components/Pages/Contracts/ContractsSearch/ContractsSearchFilter';
import ContractsSearchActions from '~/components/Pages/Contracts/ContractsSearch/ContractsSearchActions';
import ContractsSearchCard from '~/components/Pages/Contracts/ContractsSearch/ContractsSearchCard';
import ContractsSearchTabs from '~/components/Pages/Contracts/ContractsSearch/ContractsSearchTabs';

const getContracts = async({ $axios }, filter) => await $axios.get('/contracts', { params: filter });

export default {
  name: 'ContractsSearch',
  key: route => route.name,
  components: {
    SmtForm,
    ContractsSearchHeader,
    ContractsSearchFilter,
    ContractsSearchActions,
    ContractsSearchCard,
    ContractsSearchTabs,
  },
  mixins: [
    breadCrumbs,
  ],
  validate({ params }) {
    return params.page === undefined || params.page === null || /^page-([1-9]|\d{2,})$/.test(params.page);
  },
  scrollToTop: false,
  watchParam: false,
  async asyncData(context) {
    const { store, route, $moment, error } = context;
    try {
      await store.dispatch('seo/setSeoText', route.path);
      const defaultFilter = {
        CountOnPage: 20,
        TypeId: 1,
        Page: 1,
        ModeId: 1,
        OrganizerIds: [],
        ParticipantIds: [],
        StatusIds: [],
        DateSignedFrom: null,
        DateSignedTo: null,
        AmountFrom: null,
        AmountTo: null,
        Phrase: '',
        IsFavorite: false,
      };
      let contracts,
        totalCount,
        contractsFavoriteCount,
        entitiesCount;
      const filter = CloneDeep(defaultFilter);
      const setQueryToFilter = () => {
        if (route.query) {
          const { query, params } = route;
          if (query.ph) {
            filter.Phrase = query.ph;
          }
          if (query.o) {
            filter.OrganizerIds = Array.isArray(query.o) ? query.o.map(o => Number(o)) : [Number(query.o)];
          }
          if (query.pc) {
            filter.ParticipantIds = Array.isArray(query.pc) ? query.pc.map(o => Number(o)) : [Number(query.pc)];
          }
          if (query.cs) {
            filter.StatusIds = Array.isArray(query.cs) ? query.cs.map(o => Number(o)) : [Number(query.cs)];
          }
          if (query.af) {
            filter.AmountFrom = Number(query.af);
          }
          if (query.at) {
            filter.AmountTo = Number(query.at);
          }
          if (query.dsf) {
            filter.DateSignedFrom = $moment(query.dsf, 'DD-MM-YYYY').toDate();
          }
          if (query.dst) {
            filter.DateSignedTo = $moment(query.dst, 'DD-MM-YYYY').toDate();
          }
          if (query.tm) {
            filter.ModeId = Number(query.tm);
          }
          if (query.cop) {
            filter.CountOnPage = Number(query.cop);
          }
          if (params.page) {
            filter.Page = Number(/\d+/.exec(params.page)[0]);
          }
        }
      };
      setQueryToFilter();
      const setContracts = (resp) => {
        contracts = resp.data.contracts;
        totalCount = resp.data.totalCount;
        contractsFavoriteCount = resp.data.contractsFavoriteCount;
        entitiesCount = resp.data.contractsCount;
      };
      const initContracts = async(context, filter) => {
        try {
          const responseData = await getContracts(context, filter);
          setContracts(responseData);
        } catch (e) {
          error({ statusCode: e.response ? e.response.status : 500 });
        }
      };
      await initContracts(context, filter);
      await store.dispatch('referenceBook/getReferenceBooks', ['contractStatuses']);
      return {
        defaultFilter,
        filter,
        contracts,
        totalCount,
        contractsFavoriteCount,
        entitiesCount,
      };
    } catch (e) {
      error({ statusCode: e.response ? e.response.status : 500 });
    }
  },
  data() {
    return {
      tags: [],
    };
  },
  head() {
    let title = '';
    let description = '';
    if (this.filter.Page !== 1) {
      title = `${this.$t('contractsTitle')} ${this.$t('page', { number: this.filter.Page })}`;
      description = `${this.$t('contractDescription')} ${this.$t('page', { number: this.filter.Page })}`;
    } else {
      title = this.seo ? this.seo.title : this.$t('contractsTitle');
      description = this.seo ? this.seo.metaDescription : this.$t('contractDescription');
    }
    const robotsDescription = this.filter.CountOnPage !== 20 ? 'noindex, nofollow' : 'index, follow';
    return {
      title,
      meta: [
        { name: 'description', content: description },
        { name: 'robots', content: robotsDescription },
        { name: 'yandex', content: 'noindex, follow' },
      ],
      link: [
        { rel: 'canonical', href: `${this.$config.domain}${this.$route.path}` },
      ],
    };
  },
  computed: {
    ...mapState('referenceBook', [
      'referenceBooks',
    ]),
    ...mapState('seo', [
      'seo',
    ]),
    prozorroTradesLink() {
      const links = {
        uk: '/publichni-zakupivli-prozorro/',
        ru: '/ru/publichnye-zakupki-prozorro/',
        en: '/en/public-procurements-prozorro/',
      };
      return links[this.$i18n.locale];
    },
    locale() {
      if (this.$i18n.locale === 'uk') {
        return '';
      }
      return this.$i18n.locale;
    },
    breadCrumbs() {
      return [
        {
          Title: this.$t('mainPage'),
          Link: `/${this.locale}`,
        },
        {
          Title: this.$t('prozorroTrades'),
          Link: this.prozorroTradesLink,
        },
        {
          Title: this.$t('contractsProzorro'),
        },
      ];
    },
    selectedOrganizers() {
      return this.tags.filter(t => t.Type === 'OrganizerIds').map(t => ({ id: t.Id, title: t.Title, type: t.Type }));
    },
    selectedParticipants() {
      return this.tags.filter(t => t.Type === 'ParticipantIds').map(t => ({ id: t.Id, title: t.Title, type: t.Type }));
    },
    queryFilter() {
      const filter = this.filter;
      return {
        ...this.$route.query,
        ph: filter.Phrase || undefined,
        o: filter.OrganizerIds.length ? filter.OrganizerIds : undefined,
        pc: filter.ParticipantIds.length ? filter.ParticipantIds : undefined,
        cs: filter.StatusIds.length ? filter.StatusIds : undefined,
        af: filter.AmountFrom || undefined,
        at: filter.AmountTo || undefined,
        dsf: filter.DateSignedFrom ? this.$moment(filter.DateSignedFrom).format('DD-MM-YYYY') : undefined,
        dst: filter.DateSignedTo ? this.$moment(filter.DateSignedTo).format('DD-MM-YYYY') : undefined,
        cop: filter.CountOnPage !== 20 ? filter.CountOnPage : undefined,
      };
    },
    loadMoreCount() {
      const result = this.totalCount - (this.filter.Page * this.filter.CountOnPage);
      return result >= this.filter.CountOnPage ? this.filter.CountOnPage : result;
    },
    loadMoreCondition() {
      return Math.ceil(this.totalCount / this.filter.CountOnPage) !== this.filter.Page && this.totalCount;
    },
    paginationCondition() {
      return this.totalCount > this.filter.CountOnPage;
    },
    filterRules() {
      return {
        DateSignedFrom: [
          {
            shouldBeLess: this.shouldBeLess,
            active: !!this.filter.DateSignedFrom && !!this.filter.DateSignedTo,
            message: this.$t('shouldBeLess', { date: this.$moment(this.filter.DateSignedTo).format('DD.MM.YYYY') }),
          },
        ],
      };
    },
  },
  async mounted() {
    await this.getTags();
    this.getCountPageForAuthorized();
    this.setQueryFilterToRouter(this.filter.Page);
  },
  methods: {
    ...mapActions('referenceBook', [
      'getReferenceBooks',
    ]),
    changeTab(id) {
      if (id === 'all') {
        this.filter.IsFavorite = false;
      } else if (id === 'favorite') {
        this.filter.IsFavorite = true;
      }
      this.filter.Page = 1;
      this.setQueryFilterToRouter();
      getContracts(this, this.filter)
        .then(this.setContracts)
        .catch(this.setError);
    },
    changeFavoriteCount(value) {
      this.contractsFavoriteCount = this.contractsFavoriteCount += value;
    },
    setCountPageForAuthorized() {
      if (this.$auth.loggedIn) {
        localStorage.setItem('ContractsSearchCountOnPage', this.filter.CountOnPage);
      }
    },
    setFilter(filter) {
      this.filter = CloneDeep(filter);
    },
    async getTags() {
      await this.getOrganizersTags();
      await this.getParticipantsTags();
      this.getPhraseTag();
      this.getStatusesTags();
      this.getAmountTags();
      this.getDateSignedTags();
    },
    setQueryFilterToRouter(page) {
      this.$router.replace({
        query: this.queryFilter,
        params: {
          page: !page || page === 1 ? undefined : `page-${page}`,
        },
      });
    },
    scrollIntoHeader() {
      const header = document.querySelector('#contracts-header');
      header.scrollIntoView({
        block: 'start', behavior: 'smooth',
      });
    },
    setContractsMore(resp) {
      this.contracts = [...this.contracts, ...resp.data.contracts];
      this.totalCount = resp.data.totalCount;
      this.entitiesCount = resp.data.contractsCount;
    },
    setError() {
      this.$Notice.error({
        title: this.$t('error'),
        desc: this.$t('errorContracts'),
      });
    },
    getCountPageForAuthorized() {
      if (this.$auth.loggedIn && localStorage.getItem('ContractsSearchCountOnPage') !== null && this.filter.CountOnPage === 20) {
        this.filter.CountOnPage = Number(localStorage.getItem('ContractsSearchCountOnPage'));
      }
    },
    loadContracts() {
      this.filter.Page = this.filter.Page + 1;
      this.setQueryFilterToRouter(this.filter.Page);
      getContracts(this, this.filter)
        .then(this.setContractsMore)
        .catch(this.setError);
    },
    changePageSize() {
      this.filter.Page = 1;
      this.setQueryFilterToRouter();
      this.setCountPageForAuthorized();
      getContracts(this, this.filter)
        .then(this.setContracts)
        .catch(this.setError);
    },
    changeFilter() {
      this.filter.Page = 1;
      this.setQueryFilterToRouter(this.filter.Page);
      getContracts(this, this.filter)
        .then(this.setContracts)
        .catch(this.setError);
    },
    changePage(page) {
      if (this.filter.Page !== page) {
        this.filter.Page = page;
        getContracts(this, this.filter)
          .then(this.setContracts)
          .then(this.scrollIntoHeader)
          .catch(this.setError);
      }
    },
    deleteFilter(tag) {
      const filter = this.filter[tag.Type];
      const isArrayFilter = Array.isArray(filter);
      if (isArrayFilter) {
        const itemIndex = filter.findIndex(i => i === tag.Id);
        filter.splice(itemIndex, 1);
      } else {
        this.filter[tag.Type] = null;
      }
      this.changeFilter();
    },
    clearFilterAll() {
      this.setFilter(this.defaultFilter);
      this.changeFilter();
    },
    setTags(filter, type, change = true) {
      if (this.$refs.filterForm.valid()) {
        if (type === 'Phrase') {
          this.setPhraseTag(filter, type);
        } else if (type === 'OrganizerIds') {
          this.setOrganizerIdsTags(filter, type);
        } else if (type === 'ParticipantIds') {
          this.setParticipantIdsTags(filter, type);
        } else if (type === 'StatusIds') {
          this.setStatusIdsTags(filter, type);
        } else if (['AmountFrom', 'AmountTo'].includes(type)) {
          this.setAmountTag(filter, type);
        } else if (['DateSignedFrom', 'DateSignedTo'].includes(type)) {
          this.setDateSignedTag(filter, type);
        }
        if (change) {
          this.changeFilter();
        }
      }
    },
    setPhraseTag(filter, type) {
      const tagIndex = this.tags.findIndex(t => t.Type === type);
      const tag = {
        Id: 0,
        Title: filter,
        Type: type,
      };
      if (!filter) {
        this.tags.splice(tagIndex, 1);
      } else if (tagIndex !== -1) {
        this.tags.splice(tagIndex, 1, tag);
      } else {
        this.tags.push(tag);
      }
    },
    setContracts(resp) {
      this.contracts = resp.data.contracts;
      this.totalCount = resp.data.totalCount;
      this.entitiesCount = resp.data.contractsCount;
    },
    searchByOrganizers(tag, tagType) {
      this.setFilter(this.defaultFilter);
      this.deleteAllTags();
      this.filter.OrganizerIds = [tag.Id];
      this.setTags(tag, tagType);
      this.scrollIntoHeader();
    },
    searchByParticipants(tag, tagType) {
      this.setFilter(this.defaultFilter);
      this.deleteAllTags();
      this.filter.ParticipantIds = [tag.Id];
      this.setTags(tag, tagType);
      this.scrollIntoHeader();
    },
    getOrganizersTags() {
      return new Promise((resolve, reject) => {
        if (this.filter.OrganizerIds.length) {
          this.$axios.get('/ReferenceBooks/organizations', {
            params: {
              ids: this.filter.OrganizerIds,
              userType: 1,
            },
          })
            .then((resp) => {
              resp.data.forEach((o) => {
                this.tags.push({ Id: o.id, Title: o.title, Type: 'OrganizerIds' });
              });
              resolve(resp);
            })
            .catch(reject);
        } else {
          resolve();
        }
      });
    },
    getParticipantsTags() {
      return new Promise((resolve, reject) => {
        if (this.filter.ParticipantIds.length) {
          this.$axios.get('/ReferenceBooks/organizations', {
            params: {
              ids: this.filter.ParticipantIds,
              userType: 2,
            },
          })
            .then((resp) => {
              resp.data.forEach((o) => {
                this.tags.push({ Id: o.id, Title: o.title, Type: 'ParticipantIds' });
              });
              resolve(resp);
            })
            .catch(reject);
        } else {
          resolve();
        }
      });
    },
    getPhraseTag() {
      if (this.filter.Phrase) {
        this.tags.push({
          Id: 0,
          Title: this.filter.Phrase,
          Type: 'Phrase',
        });
      }
    },
    getStatusesTags() {
      if (this.filter.StatusIds) {
        this.filter.StatusIds.forEach((s) => {
          const currentStatus = this.referenceBooks.contractStatuses.find(cs => cs.id === s);
          this.tags.push({
            Id: currentStatus.id,
            Title: currentStatus.title,
            Type: 'StatusIds',
          });
        });
      }
    },
    getAmountTags() {
      if (this.filter.AmountFrom) {
        this.tags.push({
          Id: 0,
          Title: `${this.$t('amountContract')} ${this.$t('from')} ${this.filter.AmountFrom}`,
          Type: 'AmountFrom',
        });
      }
      if (this.filter.AmountTo) {
        this.tags.push({
          Id: 0,
          Title: `${this.$t('amountContract')} ${this.$t('to')} ${this.filter.AmountTo}`,
          Type: 'AmountTo',
        });
      }
    },
    getDateSignedTags() {
      if (this.filter.DateSignedFrom) {
        this.tags.push({
          Id: 0,
          Title: `${this.$t('dateSigned')} ${this.$t('from')} ${this.$moment(this.filter.DateSignedFrom).format('DD.MM.YYYY')}`,
          Type: 'DateSignedFrom',
        });
      }
      if (this.filter.DateSignedTo) {
        this.tags.push({
          Id: 0,
          Title: `${this.$t('dateSigned')} ${this.$t('to')} ${this.$moment(this.filter.DateSignedTo).format('DD.MM.YYYY')}`,
          Type: 'DateSignedTo',
        });
      }
    },
    deleteAllTags() {
      this.tags = [];
    },
    setOrganizerIdsTags(organizer, type) {
      const tagIndex = this.tags.findIndex(t => t.Type === type && t.Id === organizer.Id);
      if (tagIndex !== -1) {
        this.tags.splice(tagIndex, 1);
      } else {
        this.tags.push({
          Id: organizer.Id,
          Title: organizer.Title,
          Type: type,
        });
      }
    },
    setParticipantIdsTags(participant, type) {
      const tagIndex = this.tags.findIndex(t => t.Type === type && t.Id === participant.Id);
      if (tagIndex !== -1) {
        this.tags.splice(tagIndex, 1);
      } else {
        this.tags.push({
          Id: participant.Id,
          Title: participant.Title,
          Type: type,
        });
      }
    },
    setStatusIdsTags(filter, type) {
      const tagIndex = this.tags.findIndex(t => t.Type === type && t.Id === filter.Id);
      const currentStatus = this.referenceBooks.contractStatuses.find(s => s.id === filter.Id);
      const newTag = {
        Id: currentStatus.id,
        Title: currentStatus.title,
        Type: type,
      };
      if (filter.Value && tagIndex === -1) {
        this.tags.push(newTag);
      } else if (filter.Value && tagIndex !== -1) {
        this.tags.splice(tagIndex, 1, newTag);
      } else if (!filter.Value && tagIndex !== -1) {
        this.tags.splice(tagIndex, 1);
      }
    },
    setAmountTag(filter, type) {
      const tagIndex = this.tags.findIndex(t => t.Type === type);
      const tag = {
        Id: 0,
        Title: `${this.$t('amountContract')} ${type === 'AmountFrom' ? this.$t('from') : this.$t('to')} ${filter}`,
        Type: type,
      };
      if (!filter && tagIndex !== -1) {
        this.tags.splice(tagIndex, 1);
      } else if (tagIndex !== -1) {
        this.tags.splice(tagIndex, 1, tag);
      } else {
        this.tags.push(tag);
      }
    },
    setDateSignedTag(filter, type) {
      const tagIndex = this.tags.findIndex(t => t.Type === type);
      const tag = {
        Id: 0,
        Title: `${this.$t('dateSigned')} ${type === 'DateSignedFrom' ? this.$t('from') : this.$t('to')} ${this.$moment(filter).format('DD.MM.YYYY')}`,
        Type: type,
      };
      if (!filter && tagIndex !== -1) {
        this.tags.splice(tagIndex, 1);
      } else if (tagIndex !== -1) {
        this.tags.splice(tagIndex, 1, tag);
      } else {
        this.tags.push(tag);
      }
    },
    shouldBeLess({ value, model }) {
      return model && this.$moment(value).isSameOrBefore(this.$moment(model.DateSignedTo));
    },
  },
  nuxtI18n: {
    paths: {
      uk: '/prozorro/dogovory-zakupivel/:page?',
      ru: '/prozorro/dogovory-zakupok/:page?',
      en: '/prozorro/procurements-contracts/:page?',
    },
  },
  i18n: {
    messages: {
      uk: {
        mainPage: 'SmartTender',
        contracts: 'Договори',
        contractsProzorro: 'Договори закупівель Prozorro',
        prozorroTrades: 'Державні закупівлі PROZORRO',
        contractsTitle: 'Договори комерційних торгів та закупівель Prozorro - електронний майданчик торгів в Україні | SmartTender |',
        contractDescription: '【Повна база договорів комерційних торгів та закупівель Prozorro】. ⏩  Договори на майданчику відкритих тендерів, торгів та аукціонів в Україні ⭐ Smarttender.biz |',
        page: 'Сторінка-{number}',
        nothingFound: 'Нічого не знайдено',
        dateSigned: 'Дата підписання',
        from: 'з',
        to: 'до',
        amountContract: 'Сума контракту',
        loadMore: 'Завантажити ще {count}',
        error: 'Помилка',
        errorContracts: 'Не вдалося отримати список контрактів',
        shouldBeLess: '"з": значення має бути менше {date}',
      },
      ru: {
        mainPage: 'SmartTender',
        contracts: 'Договоры',
        contractsProzorro: 'Договоры закупок Prozorro',
        prozorroTrades: 'Публичные закупки PROZORRO',
        contractsTitle: 'Договоры коммерческих торгов и закупок Prozorro - электронная площадка торгов в Украине | SmartTender |',
        contractDescription: '【Полная база договоров коммерческих торгов и закупок Prozorro】. ⏩ Договоры на площадке открытых тендеров, торгов и аукционов в Украине ⭐ Smarttender.biz |',
        page: 'Страница-{number}',
        nothingFound: 'Ничего не найдено',
        dateSigned: 'Дата подписания',
        from: 'с',
        to: 'по',
        amountContract: 'Сумма контракта',
        loadMore: 'Загрузить еще {count}',
        error: 'Ошибка',
        errorContracts: 'Не удалось получить список контрактов',
        shouldBeLess: '"с": значение должно быть меньше {date}',
      },
      en: {
        mainPage: 'SmartTender',
        contracts: 'Contracts',
        contractsProzorro: 'Prozorro procurement contracts',
        prozorroTrades: 'Public procurements PROZORRO',
        contractsTitle: 'Prozorro commercial bidding and procurement contracts - the electronic trading platform in Ukraine | SmartTender |',
        contractDescription: '【A complete database of Prozorro commercial bidding and procurement contracts】. ⏩ The contracts for open tenders, bidding and auctions in Ukraine ⭐ Smarttender.biz |',
        page: 'Page-{number}',
        nothingFound: 'Nothing found',
        dateSigned: 'Date of signing',
        from: 'from',
        to: 'to',
        amountContract: 'Contract amount',
        loadMore: 'Load {count} more',
        error: 'Error',
        errorContracts: 'Failed to get contract list',
        shouldBeLess: '"from": value must be less {date}',
      },
    },
  },
};
</script>
<style lang="sass">
.contracts
  .contracts-container
    display: flex
  .contracts-container_block
    width: 100%
    position: relative
    perspective: 1000px
  .contract-card
    transform-origin: 50% 50% -150px
  .contracts-nothing_found
    padding: 29px 39px
    font-weight: bold
    font-size: 18px
  .block_pagination
    margin: 40px auto 20px
  .contracts-button_more
    margin: 40px auto
    font-weight: bold
@media (max-width: 1200px)
  .contracts
    .contracts-container
      flex-direction: column
@media (max-width: 767px)
  .contracts
    .contracts-nothing_found
      text-align: center
</style>
