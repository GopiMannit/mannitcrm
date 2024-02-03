<template>
  <div class="full">
    <Header class="Header" />
    <div class="search-input">
      <div class="filter-search">
        <label for="filterBy">Filter by:</label>
        <select id="filterBy" v-model="filterKey" class="filter" @change="updateFilterPlaceholder">
          <option value="">Select</option>
          <option v-for="column in tableColumns" :key="column.key" :value="column.key">{{ column.label }}</option>
        </select>
        <input type="text" class="custom-input" v-model="filterText" :placeholder="filterPlaceholder"
          :maxlength="filterKey === 'phonenumber' || filterKey === 'expectedDate' || filterKey === 'followUpDate' ? 10 : undefined" />
      </div>
    </div>
    <div class="top">
      <!-- <div class="expected-date"> -->
      <label for="date" class="label-date"> Enquiry Date :
        <input type="date" id="chooseDate" v-model="searchDate" @input="updateDate" class="date" />
        <span class="search-icon" @click="searchReport">&#128269;</span>
      </label>
      <div v-if="errorMessage" class="errorMessage">{{ errorMessage }}</div>
      <!-- </div> -->
    </div>
    <div class="table-container" id="no-more-tables">
      <table class="table">
        <thead>
          <tr>
            <th>Name</th>
            <th>PhoneNumber</th>
            <th>Gender</th>
            <th>Location</th>
            <th>Complaint</th>
            <th>Doctor Selection</th>
            <th>Finance</th>
            <th>Distance</th>
            <th>Treatment</th>
            <th>Expected Date</th>
            <th>Follow-up Date</th>
          </tr>
        </thead>
        <tbody>
          <tr v-if="source.length === 0">
            <td colspan="11" class="no-data-message">No data available</td>
          </tr>
          <tr v-else v-for="(item, index) in paginatedItems" :key="index">
            <td>{{ item.name }}</td>
            <td>{{ item.phonenumber }}</td>
            <td>{{ item.gender }}</td>
            <td>{{ item.location }}</td>
            <td>{{ item.paintype }}</td>
            <td>{{ item.doctorSelection }}</td>
            <td>{{ item.finance }}</td>
            <td>{{ item.distance }}</td>
            <td>{{ item.treatment }}</td>
            <td>{{ item.expectedDate }}</td>
            <td>{{ item.followUpDate }}</td>
          </tr>
        </tbody>
      </table>
    </div>
    <nav aria-label="Page navigation">
      <ul class="pagination">
        <li class="page-item">
          <a class="page-link" @click="changePage(pagination.page - 1)" href="#" :disabled="pagination.page === 1">
            &lt;
          </a>
        </li>
        <li class="page-item" :class="{ active: pagination.page === index + 1 }" v-for="(page, index) in totalPages"
          :key="index">
          <a class="page-link" @click="changePage(index + 1)" href="#">{{ index + 1 }}</a>
        </li>
        <li class="page-item">
          <a class="page-link" @click="changePage(pagination.page + 1)" href="#"
            :disabled="pagination.page === totalPages">
            &gt;
          </a>
        </li>
      </ul>
    </nav>
    <Footer />
  </div>
</template>

<script>
import Header from '~/components/Header.vue';
import { fetchDataWithFilter, fetchDataWithoutFilter } from '~/api/api.js';
import { extractIdentifiers } from 'vue/compiler-sfc';

export default {
  components: {
    Header,
  },
  data() {
    return {
      filterPlaceholder: 'Enter filter text',
      searchDate:'',
      errorMessage: '',
      user: {
        name: "",
        objectId: '',
        domain: '',
        subdomain: '',
        loggedInUser: {},
      },
      source: [],
      pagination: {
        page: 1,
        itemsPerPage: 10,
      },
      filterKey: '',
      filterText: '',
      tableColumns: [
        { key: 'name', label: 'Name' },
        { key: 'phonenumber', label: 'PhoneNumber' },
        { key: 'gender', label: 'Gender' },
        { key: 'location', label: 'Location' },
        { key: 'paintype', label: 'Complaint' },
        { key: 'doctorSelection', label: 'Doctor Selection' },
        { key: 'finance', label: 'Finance' },
        { key: 'distance', label: 'Distance' },
        { key: 'treatment', label: 'Treatment' },
        { key: 'expectedDate', label: 'Expected Date' },
        { key: 'followUpDate', label: 'Follow-up Date' }
      ],
    };
  },
  mounted() {
    this.searchReport();
    this.initUserData();
  },
  computed: {
    paginatedItems() {
      const start = (this.pagination.page - 1) * this.pagination.itemsPerPage;
      const end = start + this.pagination.itemsPerPage;
      return this.source.slice(start, end);
    },
    totalPages() {
      return Math.ceil(this.source.length / this.pagination.itemsPerPage);
    },
  },
  methods: {
    async searchReport() {
      try {
        if (this.filterKey && this.filterText) {
          await this.fetchDataWithFilter(this.filterKey, this.filterText);
        } else {
          // If filter is not selected, make another API call without the filter
          await this.fetchDataWithoutFilter();
        }
      } catch (error) {
        console.error('Error searching data:', error);
      }
    },

    async fetchDataWithoutFilter() {
      try {
        this.validateObjectID();

        const objectId = this.user.loggedInUser.objectId;
        const domain = this.user.loggedInUser.domain;
        const subdomain = this.user.loggedInUser.subdomain;
        const formattedDate = this.formatDate(this.searchDate);

        if (formattedDate === "") {
          this.errorMessage = "Please select the Enquire Date";
          setTimeout(() => {
            this.errorMessage = '';
          }, 4000);
          return;
        }

        // Make API call without the filter using the function from apiService.js
        const dataWithoutFilter = await fetchDataWithoutFilter(objectId, domain, subdomain, formattedDate);

        // Assuming the API response structure is similar to the one with filter
        this.source = dataWithoutFilter;
        this.pagination.page = 1;
      } catch (error) {
        console.error('Error fetching data without filter:', error);
        throw error;
      }
    },
    async fetchDataWithFilter(filterKey, filterValue) {
      try {
        this.validateObjectID();

        const objectId = this.user.loggedInUser.objectId;
        const domain = this.user.loggedInUser.domain;
        const subdomain = this.user.loggedInUser.subdomain;

        if (filterKey === "expectedDate" || filterKey === "followUpDate") {

        } else if (this.searchDate === "") {
          this.errorMessage = "Please select the Enquire Date";
          setTimeout(() => {
            this.errorMessage = '';
          }, 4000);
          return;
        }

        const response = await fetchDataWithFilter(domain, subdomain, objectId, filterKey, filterValue, this.formatDate(this.searchDate));

        this.source = response.source.map(item => JSON.parse(item));
        this.pagination.page = 1;
      } catch (error) {
        console.error('Error fetching data with filter:', error);
        throw error;
      }
    },
    updateFilterPlaceholder() {
      // Set different placeholders for each filter
      switch (this.filterKey) {
        case 'name':
          this.filterText = '';
          this.searchDate = '';
          this.filterPlaceholder = 'Enter the Name';
          break;
        case 'phonenumber':
          this.filterText = '';
          this.searchDate = '';
          this.filterPlaceholder = 'Enter the PhoneNumber';
          break;
        case 'gender':
          this.filterText = '';
          this.searchDate = '';
          this.filterPlaceholder = 'Enter the Gender';
          break;
        case 'location':
          this.filterText = '';
          this.searchDate = '';
          this.filterPlaceholder = 'Enter the location';
          break;
        case 'paintype':
          this.filterText = '';
          this.searchDate = '';
          this.filterPlaceholder = 'Enter the complaint';
          break;
        case 'doctorSelection':
          this.filterText = '';
          this.searchDate = '';
          this.filterPlaceholder = 'Enter the doctor';
          break;
        case 'finance':
        case 'distance':
        case 'treatment':
          this.filterText = '';
          this.searchDate = '';
          this.filterPlaceholder = 'Enter the number';
          break;
        case 'expectedDate':
        case 'followUpDate':
          this.filterText = '';
          this.searchDate = '';
          this.filterPlaceholder = 'dd-mm-yyyy';
          break;
        default:
          this.filterPlaceholder = 'Enter filter text';
      }
    },

    changePage(page) {
      if (page >= 1 && page <= this.totalPages) {
        this.pagination.page = page;
      }
    },

    updateDate(event) {
      this.searchDate = event.target.value;
    },

    initUserData() {
      const storedUser = sessionStorage.getItem("loggedInUser");
      if (storedUser) {
        const sourceData = JSON.parse(JSON.parse(storedUser).source);
        this.user.loggedInUser = {
          name: sourceData.username,
          objectId: sourceData._id.$oid,
          domain: sourceData.domain,
          subdomain: sourceData.subdomain,
        };
        console.log("Logged-in user data:", this.user.loggedInUser);
      } else {
        console.log("No user data found in session storage.");
      }
    },

    validateObjectID() {
      if (!this.user.loggedInUser.objectId) {
        console.error('Object ID not found in the session');
        throw new Error('Object ID not found in the session');
      }
    },
    formatDate(date) {
      const parsedDate = new Date(date);
      const day = parsedDate.getDate().toString().padStart(2, '0');
      const month = (parsedDate.getMonth() + 1).toString().padStart(2, '0');
      const year = parsedDate.getFullYear();
      return `${day}-${month}-${year}`;
    },
  },
};
</script>

<style scoped>
@import '@/styles/report.css';
</style>
