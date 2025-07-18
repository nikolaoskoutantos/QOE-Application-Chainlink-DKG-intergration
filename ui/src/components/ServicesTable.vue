<template>
  <CRow>
    <CCol>
      <CCard class="mb-4">
        <CCardHeader>
          <h5 class="mb-0">Available Services</h5>
        </CCardHeader>
        <CCardBody>
          <!-- Loading State -->
          <div v-if="loading" class="text-center py-4">
            <div class="spinner-border text-primary" role="status">
              <span class="visually-hidden">Loading services...</span>
            </div>
            <p class="mt-2">Loading services from database...</p>
          </div>

          <!-- Error State -->
          <div v-else-if="error" class="alert alert-danger">
            <strong>Error loading services:</strong> {{ error }}
            <CButton 
              color="outline-primary" 
              size="sm" 
              class="ms-2" 
              @click="servicesStore.fetchServices()"
            >
              Retry
            </CButton>
          </div>

          <!-- Services Table -->
          <div v-else-if="services.length > 0">
            <CTable align="middle" class="mb-0 border" hover responsive>
              <CTableHead color="light">
                <CTableRow>
                  <CTableHeaderCell scope="col"></CTableHeaderCell>
                  <CTableHeaderCell scope="col"></CTableHeaderCell>
                  <CTableHeaderCell scope="col">ID</CTableHeaderCell>
                  <CTableHeaderCell scope="col">Service Name</CTableHeaderCell>
                  <CTableHeaderCell scope="col">Cost</CTableHeaderCell>
                  <CTableHeaderCell scope="col">Description</CTableHeaderCell>
                  <CTableHeaderCell scope="col">Actions</CTableHeaderCell>
                </CTableRow>
              </CTableHead>
              <CTableBody>
                <template v-for="service in services" :key="'row-' + service.id">
                  <CTableRow>
                    <CTableDataCell>
                      <CFormCheck 
                        :model-value="service.checked" 
                        @update:model-value="toggleServiceSelection(service)"
                      />
                    </CTableDataCell>
                    <CTableDataCell>
                      <CButton size="sm" @click="toggleExpand(service.id)">
                        {{ expanded[service.id] ? '▼' : '▶' }}
                      </CButton>
                    </CTableDataCell>
                    <CTableDataCell>{{ service.id }}</CTableDataCell>
                    <CTableDataCell>
                      <strong>{{ service.name }}</strong>
                    </CTableDataCell>
                    <CTableDataCell>
                      <span v-if="service.link_cost">{{ service.link_cost }} LINK</span>
                      <span v-else>-</span>
                    </CTableDataCell>
                    <CTableDataCell>
                      <span class="text-muted">{{ service.description || 'No description' }}</span>
                    </CTableDataCell>
                    <CTableDataCell>
                      <div class="d-flex flex-column align-items-start gap-1">
                        <div class="d-flex gap-2">
                          <CButton color="primary" size="sm" @click="invokeService(service)">Invoke</CButton>
                          <CButton color="info" size="sm" @click="fetchCID(service)">Fetch CID</CButton>
                        </div>
                        <div v-if="CIDMap[service.id]" class="text-success small mt-1">
                          CID: {{ CIDMap[service.id] }}
                        </div>
                      </div>
                    </CTableDataCell>
                  </CTableRow>

                  <CTableRow v-if="expanded[service.id]" :key="'expand-' + service.id">
                    <CTableDataCell colspan="7">
                      <div class="p-3 bg-light">
                        <strong>Service Details:</strong>
                        <ul class="mt-2">
                          <li><strong>Smart Contract ID:</strong> {{ service.smart_contract_id || 'N/A' }}</li>
                          <li><strong>Callback Wallets:</strong> {{ service.callback_wallet_addresses || 'N/A' }}</li>
                          <li v-if="service.input_parameters">
                            <strong>Input Parameters:</strong> 
                            <pre class="small">{{ JSON.stringify(service.input_parameters, null, 2) }}</pre>
                          </li>
                        </ul>
                      </div>
                    </CTableDataCell>
                  </CTableRow>
                </template>
              </CTableBody>
            </CTable>
          </div>

          <!-- Empty State -->
          <div v-else class="text-center py-4">
            <p class="text-muted">No services found in the database.</p>
            <CButton color="primary" @click="servicesStore.fetchServices()">
              Refresh
            </CButton>
          </div>
        </CCardBody>
      </CCard>
    </CCol>
  </CRow>
</template>

<script setup>
import { reactive, onMounted } from 'vue'
import { useServicesStore } from '@/stores/services'
import { storeToRefs } from 'pinia'
import { ethers } from 'ethers'

const servicesStore = useServicesStore()
const { services, loading, error } = storeToRefs(servicesStore)

const expanded = reactive({})
const CIDMap = reactive({})

// Load services from API on component mount
onMounted(async () => {
  try {
    await servicesStore.fetchServices()
    console.log('✅ Services loaded from database')
  } catch (err) {
    console.error('❌ Failed to load services:', err)
  }
})

const CONTRACT_ADDRESS = '0x9946587cd79d59737B99D6a0b5A499584Fa3863d'
const ABI = [
  "function sendRequest(string, uint8, bytes, string[], bytes[], uint64, uint32) external",
  "function s_lastResponse() public view returns (bytes memory)"
]

// JS source used in the Chainlink request
const jsSource = `
const service = args[0];
const lat = args[1];
const lon = args[2];

const url = "https://23b6-2a02-2149-8b5d-f00-a14e-2170-6067-190e.ngrok-free.app/";

const body = {
  id: "string",
  data: {
    service: service,
    lat: parseFloat(lat),
    lon: parseFloat(lon)
  }
};

const response = await Functions.makeHttpRequest({
  url: url,
  method: "POST",
  headers: {
    "Content-Type": "application/json"
  },
  data: body
});

if (response.error) {
  throw Error(\`Request failed: \${response.error}\`);
}

const cid = response.data?.cid;

if (!cid || typeof cid !== 'string') {
  throw Error("No valid 'cid' found in the response.");
}

return Functions.encodeString(cid);
`

function toggleExpand(id) {
  expanded[id] = !expanded[id]
}

async function invokeService(service) {
  try {
    if (!window.ethereum) throw new Error("No wallet installed")
    await window.ethereum.request({ method: 'eth_requestAccounts' })

    const provider = new ethers.providers.Web3Provider(window.ethereum)
    const signer = provider.getSigner()
    const contract = new ethers.Contract(CONTRACT_ADDRESS, ABI, signer)

    const args = [ "", "48.2", "16.3"]

    const tx = await contract.sendRequest(
      jsSource,
      0,             // Location (0 = Inline)
      "0x",          // Empty encryptedSecretsReference
      args,
      [],            // bytesArgs
      344,           // Subscription ID
      300000         // Callback gas limit
    )
    await tx.wait(2)
    alert(`Request submitted to Chainlink for ${service.name}`)
  } catch (err) {
    console.error(err)
    alert('Invocation failed: ' + err.message)
  }
}

async function fetchCID(service) {
  try {
    if (!window.ethereum) throw new Error("No wallet installed")
    const provider = new ethers.providers.Web3Provider(window.ethereum)
    const signer = provider.getSigner()
    const contract = new ethers.Contract(CONTRACT_ADDRESS, ABI, signer)

    const cidBytes = await contract.s_lastResponse()
    const buffer = ethers.utils.arrayify(cidBytes)
    const cid = new TextDecoder().decode(buffer)
    CIDMap[service.id] = cid
  } catch (err) {
    console.error(err)
    alert('Fetching CID failed: ' + err.message)
  }
}

// Toggle service selection
function toggleServiceSelection(service) {
  servicesStore.toggleChecked(service.id)
}
</script>

<style scoped>
ul {
  padding-left: 1rem;
}
.text-success {
  font-weight: 600;
  color: #2e7d32;
}
</style>
