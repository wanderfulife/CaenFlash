<template>
  <div>
    <button class="btn-signin" @click="showSignInModal = true">
      Connexion Admin
    </button>

    <!-- Sign In Modal -->
    <div v-if="showSignInModal" class="modal">
      <div class="modal-content signin-modal">
        <button class="close" @click="showSignInModal = false">&times;</button>
        <h2>Connexion Admin</h2>
        <input
          v-model="password"
          type="password"
          placeholder="Entrez le mot de passe"
          class="form-control"
        />
        <button @click="signIn" class="btn-signin">Se connecter</button>
      </div>
    </div>

    <!-- Admin Dashboard Modal -->
    <div v-if="showAdminDashboard" class="modal">
      <div class="modal-content admin-dashboard">
        <button class="close" @click="showAdminDashboard = false">
          &times;
        </button>
        <h2>Tableau de Bord Admin</h2>

        <!-- Toggle Button -->
        <button @click="toggleFilter" class="btn-toggle">
          {{ showOnlyToday ? 'Voir Tout' : 'Voir Aujourd\'hui Seulement' }}
        </button>

        <div class="dashboard-content">
          <div class="dashboard-card total">
            <h3>Total des Enquêtes ({{ showOnlyToday ? 'Aujourd\'hui' : 'Tout' }})</h3>
            <p class="big-number">{{ totalSurveys }}</p>
          </div>
          <div class="dashboard-card">
            <h3>Enquêtes par Enquêteur ({{ showOnlyToday ? 'Aujourd\'hui' : 'Tout' }})</h3>
            <ul>
              <li v-for="(count, name) in surveysByEnqueteur" :key="name">
                <span>{{ name }}</span>
                <span class="count">{{ count }}</span>
              </li>
            </ul>
          </div>
          <div class="dashboard-card">
            <h3>Enquêtes par Type ({{ showOnlyToday ? 'Aujourd\'hui' : 'Tout' }})</h3>
            <ul>
              <li v-for="(count, type) in surveysByType" :key="type">
                <span>{{ type }}</span>
                <span class="count">{{ count }}</span>
              </li>
            </ul>
          </div>
        </div>
        <button @click="downloadData" class="btn-download">
          Télécharger les Données ({{ showOnlyToday ? 'Aujourd\'hui' : 'Tout' }})
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from "vue";
import { collection, getDocs } from "firebase/firestore";
import { db } from "../firebaseConfig";
import * as XLSX from "xlsx";

const showSignInModal = ref(false);
const showAdminDashboard = ref(false);
const password = ref("");
const surveysByEnqueteur = ref({});
const surveysByType = ref({});
const totalSurveys = ref(0);
const displayedSurveys = ref([]);
const allFetchedSurveys = ref([]);
const showOnlyToday = ref(true);

const surveyCollectionRef = collection(db, "CaenFlash");

// Stations list
const stationsList = [
  "Amboise",
  "Ancenis",
  "Angers-Saint-Laud",
  "Batz-sur-Mer",
  "Baule",
  "Blois-Chambord",
  "Chaingy-Fourneaux-Plage",
  "Chouzy",
  "La Baule-Escoublac",
  "La Baule-les-Pins",
  "La Chapelle-Saint-Mesmin",
  "La Chaussée-Saint-Victor",
  "Le Croisic",
  "Le Pouliguen",
  "Les Aubrais",
  "Limeray",
  "Menars",
  "Mer",
  "Meung-sur-Loire",
  "Montlouis-sur-Loire",
  "Nantes",
  "Noizay",
  "Onzain-Chaumont-sur-Loire",
  "Orléans",
  "Pornichet",
  "Saint-Ay",
  "Saint-Nazaire",
  "Saint-Pierre-des-Corps",
  "Saumur",
  "Suèvres",
  "Tours",
  "Veuves-Monteaux",
];

const signIn = () => {
  if (password.value === "Yamina123") {
    showSignInModal.value = false;
    showOnlyToday.value = true;
    fetchAdminData();
    showAdminDashboard.value = true;
  } else {
    alert("Mot de passe incorrect");
  }
};

// Helper function to get today's date in DD-MM-YYYY format
const getTodayDateString = () => {
  const today = new Date();
  const day = String(today.getDate()).padStart(2, '0');
  const month = String(today.getMonth() + 1).padStart(2, '0'); // Months are 0-based
  const year = today.getFullYear();
  return `${day}-${month}-${year}`;
};

const fetchAdminData = async (forceRefetch = false) => {
  try {
    // Fetch all surveys only if needed (first time or forced)
    if (allFetchedSurveys.value.length === 0 || forceRefetch) {
        console.log("Fetching all surveys from Firestore...");
        const querySnapshot = await getDocs(surveyCollectionRef);
        allFetchedSurveys.value = querySnapshot.docs.map((doc) => ({ id: doc.id, ...doc.data() })); // Include Firestore ID if needed
        console.log("Fetched all surveys:", allFetchedSurveys.value.length);
    }

    let surveysToDisplay = [];
    if (showOnlyToday.value) {
      // Filter for today's surveys
      const todayString = getTodayDateString();
      console.log("Filtering for date:", todayString);
      surveysToDisplay = allFetchedSurveys.value.filter(survey => survey.DATE === todayString);
      console.log("Displaying surveys from today:", surveysToDisplay.length);
    } else {
        console.log("Displaying all surveys.");
        surveysToDisplay = [...allFetchedSurveys.value]; // Use all fetched surveys
    }

    displayedSurveys.value = surveysToDisplay;

    // Calculate stats based on the displayed surveys
    totalSurveys.value = displayedSurveys.value.length;

    surveysByEnqueteur.value = displayedSurveys.value.reduce((acc, survey) => {
      const enqueteurName = survey.ENQUETEUR || "Non défini";
      acc[enqueteurName] = (acc[enqueteurName] || 0) + 1;
      return acc;
    }, {});

    surveysByType.value = displayedSurveys.value.reduce((acc, survey) => {
      let type = "Inconnu";
      if (survey.Q1 === 1) {
        type = "Voyageur";
      } else if (survey.Q1 === 2) {
        type = "Non-voyageur";
      } else {
        type = survey.TYPE_QUESTIONNAIRE || "Non-voyageur";
      }
      acc[type] = (acc[type] || 0) + 1;
      return acc;
    }, {});

  } catch (error) {
    console.error("Erreur lors de la récupération/filtrage des données :", error);
    alert(`Erreur lors de la récupération des données: ${error.message}`);
  }
};

// Function to toggle the filter and refresh data
const toggleFilter = () => {
    showOnlyToday.value = !showOnlyToday.value; // Toggle the state
    fetchAdminData(); // Refresh the dashboard data based on the new state
};

const downloadData = async () => {
  // Data to export is based on the currently displayed surveys
  const dataToExport = displayedSurveys.value;

  if (!dataToExport || dataToExport.length === 0) {
      const message = showOnlyToday.value ? "Aucune enquête trouvée pour aujourd'hui à exporter." : "Aucune enquête trouvée à exporter.";
      alert(message);
      return;
  }

  try {
    // Define header order
    const headerOrder = [
      "ID_questionnaire", "ENQUETEUR", "DATE", "JOUR", "HEURE_DEBUT", "HEURE_FIN",
      "TYPE_QUESTIONNAIRE", "Q1"
      // Add other headers if necessary
    ];

    // Map the displayed data for export
    const mappedData = dataToExport.map((docData) => {
      let type = "Inconnu";
        if (docData.Q1 === 1) {
            type = "Voyageur";
        } else if (docData.Q1 === 2) {
            type = "Non-voyageur";
        } else {
            type = docData.TYPE_QUESTIONNAIRE || "Non-voyageur";
        }
        const processedData = { ...docData, TYPE_QUESTIONNAIRE: type };

      return headerOrder.reduce((acc, key) => {
        acc[key] = processedData[key] ?? "";
        return acc;
      }, {});
    });

    const worksheet = XLSX.utils.json_to_sheet(mappedData, { header: headerOrder });

    // Set column widths
    const colWidths = headerOrder.map((header) => {
       if (header === "ID_questionnaire") return { wch: 20 };
      return { wch: 15 }; // Default width
    });
    worksheet["!cols"] = colWidths;

    const workbook = XLSX.utils.book_new();
    const sheetName = showOnlyToday.value ? "Survey Data Today" : "Survey Data All";
    XLSX.utils.book_append_sheet(workbook, worksheet, sheetName);

    // Adjust filename based on filter
    const timestamp = new Date().toTimeString().split(' ')[0].replace(/:/g, "-"); // HH-MM-SS
    let filenameBase = "CaenFlash_Survey_Data";
    if (showOnlyToday.value) {
        const todayString = getTodayDateString();
        filenameBase += `_${todayString}`;
    }
    filenameBase += `_All_${timestamp}`;
    if (!showOnlyToday.value) {
       filenameBase = `CaenFlash_Survey_Data_All_${timestamp}.xlsx`;
    }
    else{
        filenameBase = `CaenFlash_Survey_Data_${getTodayDateString()}_${timestamp}.xlsx`;
    }

    XLSX.writeFile(workbook, filenameBase);

    console.log(`File downloaded successfully with ${showOnlyToday.value ? 'today\'s' : 'all'} data`);
  } catch (error) {
    console.error("Error downloading data:", error);
    alert(`Erreur lors du téléchargement des données: ${error.message}`);
  }
};

onMounted(() => {
  // Initialization logic if needed
});
</script>

<style scoped>
.btn-signin {
  background-color: #4caf50;
  color: #ffffff;
  border: none;
  cursor: pointer;
  font-size: 16px;
  font-weight: bold;
  padding: 12px 24px;
  border-radius: 30px;
  transition: all 0.3s ease;
  text-transform: uppercase;
  letter-spacing: 1px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  margin-bottom: 20px;
}

.btn-signin:hover {
  background-color: #45a049;
  box-shadow: 0 6px 8px rgba(0, 0, 0, 0.15);
}

/* Keep the rest of the styles unchanged */
.btn-download {
  background-color: #3498db;
  color: white;
  border: none;
  padding: 10px 20px;
  border-radius: 5px;
  cursor: pointer;
  font-size: 16px;
  transition: background-color 0.3s;
  width: 100%;
  margin-top: 20px;
}

.btn-download:hover {
  background-color: #2980b9;
}

.modal {
  position: fixed;
  z-index: 1000;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.7);
  display: flex;
  justify-content: center;
  align-items: center;
}

.modal-content {
  background-color: #2d3748;
  color: white;
  padding: 25px;
  border-radius: 15px;
  width: 90%;
  position: relative;
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.4);
  max-height: fit-content;
  max-width: 500px;
  margin: 0 auto;
}

.signin-modal {
  max-width: 320px;
  padding: 20px;
}

.signin-modal h2 {
  margin: 0 0 20px 0;
  font-size: 20px;
  text-align: center;
  font-weight: normal;
}

.signin-modal .form-control {
  width: 100%;
  margin-bottom: 15px;
  background-color: rgba(255, 255, 255, 0.1);
  border: none;
  color: white;
  padding: 12px;
  font-size: 14px;
  border-radius: 8px;
  box-sizing: border-box;
}

.signin-modal .form-control::placeholder {
  color: rgba(255, 255, 255, 0.6);
}

.signin-modal .btn-signin {
  width: 100%;
  margin: 0;
  padding: 12px;
  background-color: #68d391;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 14px;
  font-weight: bold;
  text-transform: uppercase;
  cursor: pointer;
  transition: background-color 0.3s;
}

.signin-modal .btn-signin:hover {
  background-color: #5cb67e;
}

.close {
  position: absolute;
  right: 15px;
  top: 15px;
  width: 24px;
  height: 24px;
  opacity: 0.7;
  background: none;
  border: none;
  cursor: pointer;
  color: white;
  font-size: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: opacity 0.3s;
}

.close:hover {
  opacity: 1;
}

.admin-dashboard {
  max-width: 500px;
  padding: 20px 30px;
  height: auto;
}

.admin-dashboard h2 {
  margin: 0 0 20px 0;
  font-size: 24px;
  text-align: center;
  font-weight: normal;
  color: white;
}

.dashboard-content {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-bottom: 20px;
}

.dashboard-card {
  background-color: rgba(255, 255, 255, 0.1);
  border-radius: 10px;
  padding: 15px;
}

.dashboard-card h3 {
  margin: 0 0 8px 0;
  font-size: 16px;
  color: #3b82f6;
  font-weight: normal;
}

.dashboard-card.total {
  text-align: center;
  padding: 12px;
}

.big-number {
  font-size: 42px;
  font-weight: bold;
  color: #68d391;
  margin: 5px 0;
}

.dashboard-card ul {
  list-style-type: none;
  padding: 0;
  margin: 0;
}

.dashboard-card li {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 6px 0;
  color: white;
}

.count {
  font-weight: normal;
  color: #68d391;
}

.btn-download {
  width: 100%;
  padding: 12px;
  background-color: #3b82f6;
  color: white;
  border: none;
  border-radius: 10px;
  font-size: 16px;
  font-weight: normal;
  cursor: pointer;
  transition: background-color 0.3s;
  margin-top: 0;
}

.btn-download:hover {
  background-color: #2563eb;
}

.btn-toggle {
  background-color: #f39c12; /* Orange color */
  color: white;
  border: none;
  padding: 8px 15px;
  border-radius: 5px;
  cursor: pointer;
  font-size: 14px;
  margin-bottom: 15px; /* Add some space below the button */
  display: block; /* Make it block to center */
  margin-left: auto;
  margin-right: auto;
  width: fit-content; /* Adjust width */
  transition: background-color 0.3s;
}

.btn-toggle:hover {
  background-color: #e67e22;
}

/* Add labels to dashboard cards */
.dashboard-card h3 {
  /* Adjust existing styles if needed, maybe add more margin-bottom */
  margin-bottom: 10px;
}

@media (max-width: 600px) {
  .modal-content {
    margin: 20px;
    width: calc(100% - 40px);
  }

  .admin-dashboard {
    padding: 20px;
  }

  .admin-dashboard h2 {
    font-size: 20px;
    margin-bottom: 15px;
  }

  .dashboard-card {
    padding: 12px;
  }

  .big-number {
    font-size: 36px;
  }
}
</style>