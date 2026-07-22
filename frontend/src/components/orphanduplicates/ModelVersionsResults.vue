<template>
  <div v-if="showModelVersions" class="model-versions-section">
    <h2 class="section-title">Model Versions</h2>
    <div v-if="modelVersionsLoading" class="loading-state">
      <div class="loading-spinner"></div>
      <span>Loading models with multiple versions...</span>
    </div>
    <div v-else-if="modelVersionsError" class="error-card">{{ modelVersionsError }}</div>
    <div v-else-if="models.length > 0" class="results-container">
      <div class="results-header">
        <h3 class="results-title">
          Showing {{ filteredModels.length }} of {{ totalModels }} models
          <span class="results-version-count">({{ filteredVersionCount }} versions)</span>
        </h3>
        <p class="results-subtitle">
          These CivitAI models have more than one version in your database. Single-version models are hidden.
        </p>

        <div class="filters-row">
          <label class="filter-item">
            <span class="filter-label">Base Model</span>
            <select v-model="selectedBaseModel" class="filter-select">
              <option value="">All base models</option>
              <option v-for="baseModel in availableBaseModels" :key="baseModel" :value="baseModel">
                {{ baseModel }}
              </option>
            </select>
          </label>

          <label class="filter-item">
            <span class="filter-label">Status</span>
            <select v-model="selectedStatus" class="filter-select">
              <option value="">All statuses</option>
              <option
                v-for="option in statusFilterOptions"
                :key="option.value"
                :value="String(option.value)"
              >
                {{ option.label }}
              </option>
            </select>
          </label>
        </div>
      </div>

      <div v-if="filteredModels.length === 0" class="no-filter-results">
        No models match the selected filters.
      </div>

      <div v-else class="table-container">
        <table class="model-versions-table">
          <thead>
            <tr>
              <th>Model</th>
              <th>Versions, Status & Base Model</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="model in filteredModels" :key="model.modelId" class="table-row">
              <td class="model-cell">
                <span class="model-name">{{ model.modelName || 'Unknown model' }}</span>
                <span class="version-count">({{ model.versionCount }} versions)</span>
              </td>
              <td class="versions-cell">
                <template v-for="(version, index) in model.versions" :key="version.modelVersionId">
                  <span class="version-group">
                    <a
                      :href="getProfileUrl(model.modelId, version.modelVersionId)"
                      target="_blank"
                      rel="noopener noreferrer"
                      class="version-link"
                      :title="version.modelVersionName || `Version ${version.modelVersionId}`"
                    >
                      {{ version.modelVersionId }}
                    </a>
                    <span class="status-badge" :class="getStatusClass(version.isDownloaded)">
                      {{ getStatusLabel(version.isDownloaded) }}
                    </span>
                    <span class="basemodel-tag">[{{ version.basemodel || 'N/A' }}]</span>
                  </span>
                  <span v-if="index < model.versions.length - 1" class="version-separator"> / </span>
                </template>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
    <div v-else class="no-results">
      <div class="no-results-icon">✅</div>
      <h3 class="no-results-title">No multi-version models found</h3>
      <p class="no-results-subtitle">Every model in your database currently has only one version</p>
    </div>
  </div>
</template>

<script>
import { DOWNLOAD_STATUS } from '@/utils/constants.js';

export default {
  name: 'ModelVersionsResults',
  props: {
    showModelVersions: {
      type: Boolean,
      default: false
    },
    modelVersionsLoading: {
      type: Boolean,
      default: false
    },
    modelVersionsError: {
      type: String,
      default: null
    },
    models: {
      type: Array,
      default: () => []
    },
    totalModels: {
      type: Number,
      default: 0
    },
    totalVersions: {
      type: Number,
      default: 0
    },
    frontendBaseUrl: {
      type: String,
      required: true
    }
  },
  data() {
    return {
      selectedBaseModel: '',
      selectedStatus: ''
    };
  },
  computed: {
    statusFilterOptions() {
      return [
        { value: DOWNLOAD_STATUS.NOT_DOWNLOADED, label: 'Not downloaded' },
        { value: DOWNLOAD_STATUS.DOWNLOADED, label: 'Downloaded' },
        { value: DOWNLOAD_STATUS.DOWNLOADING, label: 'Downloading' },
        { value: DOWNLOAD_STATUS.FAILED, label: 'Failed' },
        { value: DOWNLOAD_STATUS.IGNORED, label: 'Ignored' }
      ];
    },
    availableBaseModels() {
      const baseModels = new Set();
      this.models.forEach(model => {
        (model.versions || []).forEach(version => {
          if (version.basemodel) {
            baseModels.add(version.basemodel);
          }
        });
      });
      return Array.from(baseModels).sort((a, b) => a.localeCompare(b));
    },
    filteredModels() {
      return this.models.filter(model => this.modelMatchesFilters(model));
    },
    filteredVersionCount() {
      return this.filteredModels.reduce(
        (sum, model) => sum + (model.versions?.length || 0),
        0
      );
    }
  },
  watch: {
    models() {
      this.selectedBaseModel = '';
      this.selectedStatus = '';
    }
  },
  methods: {
    modelMatchesFilters(model) {
      const versions = model.versions || [];

      if (this.selectedBaseModel) {
        const hasBaseModel = versions.some(
          version => version.basemodel === this.selectedBaseModel
        );
        if (!hasBaseModel) return false;
      }

      if (this.selectedStatus !== '') {
        const statusValue = Number(this.selectedStatus);
        const hasStatus = versions.some(
          version => version.isDownloaded === statusValue
        );
        if (!hasStatus) return false;
      }

      return true;
    },
    getProfileUrl(modelId, modelVersionId) {
      return `${this.frontendBaseUrl}/model/${modelId}/${modelVersionId}`;
    },
    getStatusLabel(status) {
      switch (status) {
        case DOWNLOAD_STATUS.DOWNLOADED: return 'Downloaded';
        case DOWNLOAD_STATUS.DOWNLOADING: return 'Downloading';
        case DOWNLOAD_STATUS.FAILED: return 'Failed';
        case DOWNLOAD_STATUS.IGNORED: return 'Ignored';
        default: return 'Not downloaded';
      }
    },
    getStatusClass(status) {
      switch (status) {
        case DOWNLOAD_STATUS.DOWNLOADED: return 'status-downloaded';
        case DOWNLOAD_STATUS.DOWNLOADING: return 'status-downloading';
        case DOWNLOAD_STATUS.FAILED: return 'status-failed';
        case DOWNLOAD_STATUS.IGNORED: return 'status-ignored';
        default: return 'status-not-downloaded';
      }
    }
  }
};
</script>

<style scoped>
.model-versions-section {
  margin-top: 2rem;
  background: #ffffff;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  overflow: hidden;
}

.section-title {
  margin: 0;
  padding: 1.5rem 1.5rem 1rem;
  font-size: 1.5rem;
  font-weight: 600;
  color: #1a1a1a;
  border-bottom: 1px solid #e0e0e0;
}

.loading-state {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 2rem;
  color: #666;
}

.loading-spinner {
  width: 20px;
  height: 20px;
  border: 2px solid #e0e0e0;
  border-top: 2px solid #2196f3;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.error-card {
  color: #d32f2f;
  font-weight: 500;
  padding: 1rem 1.5rem;
  background: #ffebee;
  border-left: 4px solid #d32f2f;
  margin: 1rem 1.5rem;
  border-radius: 4px;
}

.results-container {
  padding: 1.5rem;
}

.results-header {
  margin-bottom: 1.5rem;
}

.results-title {
  margin: 0 0 0.5rem;
  font-size: 1.2rem;
  color: #2c3e50;
}

.results-subtitle {
  margin: 0 0 1rem;
  color: #666;
  font-size: 0.95rem;
}

.results-version-count {
  color: #666;
  font-size: 0.95rem;
  font-weight: 500;
}

.filters-row {
  display: flex;
  flex-wrap: wrap;
  gap: 1rem;
  align-items: flex-end;
}

.filter-item {
  display: flex;
  flex-direction: column;
  gap: 0.35rem;
  min-width: 200px;
}

.filter-label {
  font-size: 0.85rem;
  font-weight: 600;
  color: #495057;
}

.filter-select {
  padding: 0.55rem 0.75rem;
  border: 1px solid #ced4da;
  border-radius: 8px;
  background: #fff;
  color: #333;
  font-size: 0.9rem;
  cursor: pointer;
}

.filter-select:focus {
  outline: none;
  border-color: #1976d2;
  box-shadow: 0 0 0 2px rgba(25, 118, 210, 0.15);
}

.no-filter-results {
  padding: 2rem 1rem;
  text-align: center;
  color: #666;
  font-style: italic;
  background: #f8f9fa;
  border-radius: 8px;
}

.table-container {
  overflow-x: auto;
}

.model-versions-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.92rem;
}

.model-versions-table th,
.model-versions-table td {
  padding: 0.85rem 1rem;
  border-bottom: 1px solid #e9ecef;
  text-align: left;
  vertical-align: middle;
}

.model-versions-table th {
  background: #f8f9fa;
  font-weight: 600;
  color: #495057;
}

.model-cell {
  min-width: 220px;
}

.model-name {
  font-weight: 700;
  color: #2c3e50;
}

.version-count {
  margin-left: 0.4rem;
  color: #1976d2;
  font-size: 0.85rem;
  font-weight: 600;
}

.versions-cell {
  font-family: 'Courier New', monospace;
  font-size: 0.9rem;
  line-height: 1.8;
}

.version-group {
  display: inline-flex;
  align-items: center;
  gap: 0.35rem;
  vertical-align: middle;
}

.version-link {
  color: #1976d2;
  text-decoration: none;
  font-weight: 600;
}

.version-link:hover {
  text-decoration: underline;
}

.version-separator {
  color: #666;
  margin: 0 0.15rem;
}

.status-badge {
  display: inline-block;
  padding: 0.2rem 0.5rem;
  border-radius: 999px;
  font-size: 0.75rem;
  font-weight: 600;
  white-space: nowrap;
}

.status-downloaded {
  background: #e8f5e9;
  color: #2e7d32;
}

.status-downloading {
  background: #e3f2fd;
  color: #1565c0;
}

.status-failed {
  background: #fff3e0;
  color: #ef6c00;
}

.status-ignored {
  background: #fff8e1;
  color: #f57f17;
}

.status-not-downloaded {
  background: #f5f5f5;
  color: #616161;
}

.basemodel-tag {
  display: inline-block;
  padding: 0.05rem 0.25rem;
  background: #e9e9e9;
  color: #555;
  font-size: 0.72rem;
  font-weight: 700;
  font-family: inherit;
  white-space: nowrap;
  line-height: 1.2;
}

.no-results {
  padding: 3rem 1.5rem;
  text-align: center;
}

.no-results-icon {
  font-size: 2rem;
  margin-bottom: 0.75rem;
}

.no-results-title {
  margin: 0 0 0.5rem;
  color: #2c3e50;
}

.no-results-subtitle {
  margin: 0;
  color: #666;
}
</style>
