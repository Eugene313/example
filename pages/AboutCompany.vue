<template>
  <div class="about-company">
    <AboutCompanyHeader />
    <AboutCompanyHistory />
    <AboutCompanyStats
      :general-blocks="model.generalBlocks"
    />
    <AboutCompanyDirections />
    <AboutCompanyBenefits
      :benefits-blocks="model.benefitsBlocks"
    />
    <AboutCompanyWhy />
    <AboutCompanyServices />
    <AboutCompanyRegistration />
  </div>
</template>
<script>
import { mapState } from 'vuex';
import BreadCrumbs from '~/mixins/breadCrumbs';
import AboutCompanyHeader from '~/components/Pages/Articles/AboutCompany/AboutCompanyHeader';
import AboutCompanyHistory from '~/components/Pages/Articles/AboutCompany/AboutCompanyHistory';
import AboutCompanyStats from '~/components/Pages/Articles/AboutCompany/AboutCompanyStats';
import AboutCompanyDirections from '~/components/Pages/Articles/AboutCompany/AboutCompanyDirections';
import AboutCompanyBenefits from '~/components/Pages/Articles/AboutCompany/AboutCompanyBenefits';
import AboutCompanyWhy from '~/components/Pages/Articles/AboutCompany/AboutCompanyWhy';
import AboutCompanyServices from '~/components/Pages/Articles/AboutCompany/AboutCompanyServices';
import AboutCompanyRegistration from '~/components/Pages/Articles/AboutCompany/AboutCompanyRegistration';

export default {
  name: 'AboutCompany',
  components: {
    AboutCompanyHeader,
    AboutCompanyHistory,
    AboutCompanyStats,
    AboutCompanyDirections,
    AboutCompanyBenefits,
    AboutCompanyWhy,
    AboutCompanyServices,
    AboutCompanyRegistration,
  },
  mixins: [
    BreadCrumbs,
  ],
  async asyncData({ $axios, error, store, route }) {
    try {
      await store.dispatch('seo/setSeoText', route.path);
      await store.dispatch('referenceBook/getRegistrationUserRoles');
      const { data: model } = await $axios.get('/AboutCompanyContents');
      return { model };
    } catch (e) {
      error({ statusCode: e.response ? e.response.status : 500 });
    }
  },
  data() {
    return {
      model: {
        generalBlocks: [],
        benefitsBlocks: {
          commercialOrganizerBlocks: [],
          providerBlocks: [],
          prozorroSaleBlocks: [],
          stateOrganizerBlocks: [],
        },
      },
    };
  },
  head() {
    return {
      title: this.seo ? this.seo.title : null,
      meta: [
        { name: 'description', content: this.seo ? this.seo.metaDescription : null },
      ],
      link: [
        { rel: 'canonical', href: `${this.$config.domain}${this.$route.path}` },
      ],
    };
  },
  nuxtI18n: {
    paths: {
      uk: '/etm/pro-kompaniyu',
      ru: '/etm/o-kompanii',
      en: '/etm/about-company',
    },
  },
  computed: {
    ...mapState('seo', [
      'seo',
    ]),
    breadCrumbs() {
      return [
        { Title: this.$t('mainPage'), Link: this.$i18n.locale === 'uk' ? '/' : `/${this.$i18n.locale}` },
        { Title: this.$t('aboutSmartTender') },
      ];
    },
  },
  i18n: {
    messages: {
      uk: {
        mainPage: 'SmartTender',
        aboutSmartTender: 'Про SmartTender',
      },
      ru: {
        mainPage: 'SmartTender',
        aboutSmartTender: 'Про SmartTender',
      },
      en: {
        mainPage: 'SmartTender',
        aboutSmartTender: 'About SmartTender',
      },
    },
  },
};
</script>
<style>
.about-company {
  max-width: 100%;
  overflow-x: hidden;
}
</style>
