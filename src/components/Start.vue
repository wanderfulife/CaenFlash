<template>
  <div class="app-container">
    <!-- Progress Bar -->
    <div v-if="currentStep === 'survey'" class="progress-bar">
      <div class="progress" :style="{ width: `${progress}%` }"></div>
    </div>

    <div class="content-container">
      <!-- Enqueteur Input Step -->
      <div v-if="currentStep === 'enqueteur'">
        <h2>Prénom enqueteur :</h2>
        <input class="form-control" type="text" v-model="enqueteur" />
        <button
          v-if="enqueteur && !isEnqueteurSaved"
          @click="setEnqueteur"
          class="btn-next"
        >
          Suivant
        </button>
      </div>

      <!-- Start Survey Step -->
      <div v-else-if="currentStep === 'start'" class="start-survey-container">
        <h2>
          Bonjour,<br />
          pour mieux connaître les usagers<br />
          de la gare de Caen,<br /><br />
          la Ville et la SNCF souhaiteraient en savoir plus sur votre
          déplacement en cours.<br />
          Auriez-vous quelques secondes à nous accorder ?
        </h2>
        <button @click="startSurvey" class="btn-next">
          COMMENCER QUESTIONNAIRE
        </button>
      </div>

      <!-- Survey Questions Step -->
      <div v-else-if="currentStep === 'survey' && !isSurveyComplete">
        <div class="question-container">
          <h2>{{ currentQuestion.text }}</h2>

          <!-- PDF Button for Q3a / Q3b -->
          <button
            v-if="currentQuestion.id === 'Q3a' || currentQuestion.id === 'Q3b'"
            @click="() => {
              if (currentQuestion.id === 'Q3a') {
                pdfUrl = '/Plan.pdf';
                console.log('Setting PDF URL to /Plan.pdf');
              } else if (currentQuestion.id === 'Q3b') {
                pdfUrl = '/Plan2.pdf';
                console.log('Setting PDF URL to /Plan2.pdf');
              }
              console.log('Opening PDF modal');
              showPdf = true;
            }"
            class="btn-pdf"
          >
            Voir le plan du parking
          </button>

          <!-- Commune Selector for Q2 and Q6 -->
          <div v-if="currentQuestion.id === 'Q2'">
            <div
              v-for="(option, index) in currentQuestion.options"
              :key="index"
            >
              <button @click="selectAnswer(option, index)" class="btn-option">
                {{ option.text }}
              </button>
            </div>
          </div>
          <div
            v-else-if="
              currentQuestion.id === 'Q2_precision' ||
              currentQuestion.id === 'Q6'
            "
          >
            <CommuneSelector
              v-model="selectedCommune"
              v-model:postalCodePrefix="postalCodePrefix"
            />
            <p>Commune sélectionnée ou saisie: {{ selectedCommune }}</p>
            <button
              @click="handleCommuneSelection"
              class="btn-next"
              :disabled="!selectedCommune.trim()"
            >
              {{ isLastQuestion ? "Terminer" : "Suivant" }}
            </button>
          </div>

          <!-- Street Input -->
          <div v-else-if="currentQuestion.streetInput">
            <div class="input-container">
              <input
                v-model="streetInput"
                class="form-control"
                type="text"
                :placeholder="
                  currentQuestion.freeTextPlaceholder || 'Saisissez une rue'
                "
              />
              <ul v-if="showFilteredStreets" class="commune-dropdown">
                <li
                  v-for="street in filteredStreets"
                  :key="street"
                  @click="selectStreet(street)"
                  class="commune-option"
                >
                  {{ street }}
                </li>
              </ul>
            </div>
            <button
              @click="handleStreetSelection"
              class="btn-next"
              :disabled="!streetInput.trim()"
            >
              {{ isLastQuestion ? "Terminer" : "Suivant" }}
            </button>
          </div>
          <!-- Multiple Choice Questions -->
          <div
            v-else-if="
              !currentQuestion.freeText && !currentQuestion.usesGareSelector
            "
          >
            <div
              v-for="(option, index) in currentQuestion.options"
              :key="index"
            >
              <button @click="selectAnswer(option, index)" class="btn-option">
                {{ option.text }}
              </button>
            </div>
          </div>
          <!-- Gare Selector -->
          <div v-else-if="currentQuestion.usesGareSelector">
            <GareSelector v-model="selectedStation" />
            <button
              @click="handleStationSelection"
              class="btn-next"
              :disabled="!selectedStation.trim()"
            >
              {{ isLastQuestion ? "Terminer" : "Suivant" }}
            </button>
          </div>
          <!-- Free Text Questions -->
          <div v-else>
            <div class="input-container">
              <input
                v-model="freeTextAnswer"
                class="form-control"
                type="text"
                :placeholder="
                  currentQuestion.freeTextPlaceholder || 'Votre réponse'
                "
              />
            </div>
            <button
              @click="handleFreeTextAnswer"
              class="btn-next"
              :disabled="!freeTextAnswer.trim()"
            >
              {{ isLastQuestion ? "Terminer" : "Suivant" }}
            </button>
          </div>
          <!-- Back Button -->
          <button @click="previousQuestion" class="btn-return" v-if="canGoBack">
            Retour
          </button>
          <!-- Partial Validation Button -->
        </div>
      </div>

      <!-- Survey Complete Step -->
      <div v-else-if="isSurveyComplete" class="survey-complete">
        <h2>Merci pour votre réponse et bonne journée.</h2>
        <button @click="resetSurvey" class="btn-next">
          Nouveau questionnaire
        </button>
      </div>

      <!-- Logo -->
      <img class="logo" src="../assets/Alycelogo.webp" alt="Logo Alyce" />
    </div>

    <!-- Footer -->
    <div class="footer">
      <AdminDashboard />
    </div>

    <!-- PDF Modal -->
    <div v-if="showPdf" class="modal">
      <div class="modal-content pdf-content">
        <button class="close" @click="() => { showPdf = false; console.log('Closing PDF modal'); }">
          ×
        </button>
        <iframe :src="pdfUrl" width="100%" height="500px" type="application/pdf">
          This browser does not support PDFs. Please download the PDF to view it.
        </iframe>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch } from "vue";
import { db } from "../firebaseConfig";
import { collection, getDocs, addDoc } from "firebase/firestore";
import { doc, getDoc, setDoc } from "firebase/firestore";
import { questions } from "./surveyQuestions.js";
import CommuneSelector from "./CommuneSelector.vue";
import AdminDashboard from "./AdminDashboard.vue";
import GareSelector from "./GareSelector.vue";

// Refs
const docCount = ref(0);
const currentStep = ref("enqueteur");
const startDate = ref("");
const enqueteur = ref("");
const currentQuestionIndex = ref(0);
const answers = ref({});
const freeTextAnswer = ref("");
const questionPath = ref(["Q1"]);
const isEnqueteurSaved = ref(false);
const isSurveyComplete = ref(false);
const selectedStation = ref("");
const selectedCommune = ref("");
const postalCodePrefix = ref("");
const showPdf = ref(false);
const pdfUrl = ref("/Plan.pdf");
const stationInput = ref("");
const streetInput = ref("");
const filteredStations = ref([]);
const filteredStreets = ref([]);
const selectedPoste = ref(null);

// Firestore refs
const surveyCollectionRef = collection(db, "CaenFlash");
const counterDocRef = doc(db, "CaenFlash", "surveyCounter");

// Stations list

const stationsList = [
  "Amboise",
  "Ancenis",
  "La Angers-Saint-Laud-Landerneau",
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

const streetsList = [
  "11 novembre (Rue du)",
  "2 Siciles (Rue des)",
  "36e Régiment d'Infanterie (Place du)",
  "43e Régiment d'Artillerie (Avenue du)",
  "6 juin (Avenue du)",
  "87 Fusillés (Rond-Point des)",
  "Abbatiale (Rue de l')",
  "Abbaye d'Ardenne (Chemin de l')",
  "Abbaye d'Ardennes (Rue de l')",
  "Abbé de la Rue (Rue)",
  "Abricotiers (Allée des)",
  "Acacia (Allée de l')",
  "Académie (Rue de l')",
  "Acadiens (Rue des)",
  "Albert Einstein (Rue)",
  "Albert Premier (Avenue)",
  "Albert Sorel (Avenue)",
  "Albert de Mun (Rue)",
  "Alexandre (Rue)",
  "Alexandre Bigot (Rue)",
  "Alexandre Dumas (Rue)",
  "Alexandria (Rue, d')",
  "Alfred Kastler (Rue)",
  "Alfred Nobel (Rue)",
  "Alfred de Musset (Rue)",
  "Alger (Chaussée d')",
  "Alliés (Boulevard des)",
  "Alouettes (Rue des)",
  "Alphonse et Léonard Gille (Rue)",
  "Alsace (Rue d')",
  "Amand Bence (Rue)",
  "Amblie (Rue d')",
  "Amiral Hamelin (Quai)",
  "Amiral Mountbatten (Avenue)",
  "Ampère (Rue)",
  "Anatole Lelièvre (Rue)",
  "Ancienne Boucherie (Place de l')",
  "Ancienne Comédie (Impasse de l')",
  "Ancienne Comédie (Place de l')",
  "Anciens d'Afrique Française du Nord (Rue des)",
  "André Chapron (Impasse)",
  "André Detolle (Boulevard)",
  "Andre Malraux (Esplanade)",
  "Anémones (Rue des)",
  "Anisy (Rue d')",
  "Antoine Cavelier (Rue)",
  "Antoine Galland (Rue)",
  "Arago (Rue)",
  "Arc (Chemin de l')",
  "Arc (Résidence, de l')",
  "Arcisse de Caumont (Rue)",
  "Ardennes (Rue d')",
  "Aristide Briand (Boulevard)",
  "Arlette de Falaise (Rue)",
  "Armand Marie (Rue)",
  "Armor (Rue d')",
  "Arquette (Rue de l')",
  "Arthur Le Duc (Rue)",
  "Arts (Allées des)",
  "Ateliers (Rue des)",
  "Aubépines (Allée des)",
  "Aubusson (Rue d')",
  "Auderville (Allée d')",
  "Auge (Rue d')",
  "Auguste Blanc (Rue)",
  "Auguste Lechesne (Rue)",
  "Auguste Nicolas (Rue)",
  "Aurigny (Rue d')",
  "Aurore (Rue de l')",
  "Authie (Rue d')",
  "Auvergne (Rue d')",
  "Avelines (Allée des)",
  "Aviation (Rue de l')",
  "B du Mont Coco (Impasse)",
  "B du Mont Coco (Impasse)",
  "Bagatelle (Avenue de)",
  "Bagatelle (Impasse de)",
  "Bailey (Rue)",
  "Bailliage (Rue du)",
  "Baladas (Boulevard des)",
  "Balzac (Rue)",
  "Barbeux (Rue)",
  "Barbey d'Aurevilly (Rue)",
  "Basse (Rue)",
  "Baudelaire (Allée)",
  "Baumier (Rue)",
  "Bayeux (Rue de)",
  "Beau Regard (Rue)",
  "Beau Site (Chemin du)",
  "Beau Site (Rue du)",
  "Beau-Soleil (Rue)",
  "Beaufort (Allée de)",
  "Beaulieu (Rue de)",
  "Beauséjour (Allée)",
  "Beausejour (Rue)",
  "Bec Hellouin (Allée du)",
  "Bel-Air (Rue du)",
  "Bellevue (Rue de)",
  "Bellivet (Rue)",
  "Bélvédère (Impasse du)",
  "Belvédère (Rue)",
  "Bénédic Macé (Rue)",
  "Bénédictins (Rue des)",
  "Bengale (Rue, du)",
  "Bénouville (Rue de)",
  "Bény-sur-Mer (Rue de)",
  "Bernard Palissy (Rue)",
  "Bernard Vanier (Rue)",
  "Bernières (Rue de)",
  "Berry (Rue de)",
  "Bertauld (Rue)",
  "Bertrand (Boulevard)",
  "Bessin (Rue, du)",
  "Beuville (Rue de)",
  "Beuvrelu (Impasse)",
  "Beuvrelu (Rue)",
  "Bicoquet (Rue)",
  "Bief (Passage du)",
  "Bienfaisance (Rue de la)",
  "Biéville (Rue de)",
  "Bissonnet (Rue du)",
  "Blainville (Rue de)",
  "Blanc (Rue du)",
  "Blanc Hardel (Rue le)",
  "Blanche Herbe (Rue de la)",
  "Blanchisseries (Rue des)",
  "Blériot (Rue)",
  "Bleuets (Rue des)",
  "Blot (Place)",
  "Bocage (Allée du)",
  "Bois Robert (Rue de)",
  "Bon Accueil (Impasse du)",
  "Bons Enfants (Rue des)",
  "Bosnières (Rue)",
  "Bosphore (Allée du)",
  "Bouleaux (Allée, des)",
  "Bourgogne (Rue de)",
  "Boutiques (Rue des)",
  "Bouviers (Rue des)",
  "Bouvreuil (Rue du)",
  "Branly (Rue)",
  "Branville (Rue)",
  "Bras (Rue de)",
  "Brébeufs (Chemin des)",
  "Brécy (Rue de)",
  "Brest (Boulevard de)",
  "Bretagne (Rue de)",
  "Bretteville (Impasse de)",
  "Bretteville (Rue de)",
  "Bretteville Odon (Chemin)",
  "Brillaud de Laujardière (Esplanade)",
  "Broceliande (Rue de)",
  "Bruxelles (Avenue de)",
  "Buquet (Rue)",
  "Buron (Rue de)",
  "Bœufs (Chemin aux)",
  "Caffarelli (Cours)",
  "Caffarelli (Quai)",
  "Cairon (Rue de)",
  "Calibourg (Rue)",
  "Calix (Rue de)",
  "Callu (Impasse)",
  "Calvados (Avenue du)",
  "Cambes (Rue de)",
  "Camille Guérin (Rue)",
  "Campion (Rue)",
  "Canada (Avenue du)",
  "Canada (Place du)",
  "Canchy (Rue)",
  "Capitaine Boualam (Rue)",
  "Capitaine Foucher (Rue du)",
  "Capitaine Georges Guynemer (Avenue du)",
  "Caponière (Rue)",
  "Capucines (Rue des)",
  "Cardiff (Rue de)",
  "Cardinal Lavigerie (Rue)",
  "Cardonnière (Rue de la)",
  "Carel (Rue du)",
  "Carmélites (Rue des)",
  "Carmes (Rue des)",
  "Carpiquet (Chemin de)",
  "Carrières Saint-Julien (Rue des)",
  "Carrières de Vaucelles (Rue des)",
  "Cauvigny (Rue, de)",
  "Cécile de Normandie (Allée)",
  "Cèdres (Allée des)",
  "Cerisiers (Rue des)",
  "Champagne (Rue de)",
  "Champlain (Place)",
  "Champs (Venelle aux)",
  "Champs Saint-Michel (Rue des)",
  "Chanoine Cousin (Passage)",
  "Chanoine Delamazure (Rue)",
  "Chanoine Ruel (Rue du)",
  "Chanoine Vautier (Rue)",
  "Chanoine Xavier de Saint-Pol (Rue)",
  "Chanoines (Rue des)",
  "Chapelle (Rue de la)",
  "Chardonneret (Résidence, du)",
  "Chardonneret (Rue du)",
  "Charité (Boulevard de la)",
  "Charlemagne (Avenue)",
  "Charles Lamusse (Promenade)",
  "Charles Léandre (Rue)",
  "Charles Lemaître (Rue)",
  "Charles Péguy (Rue)",
  "Charles Prunier (Rue)",
  "Charlotte Corday (Avenue)",
  "Château d'Eau (Rue du)",
  "Chateaubriand (Rue)",
  "Chaussée Ferrée (Rue de la)",
  "Chemin Fourchu (Rue du)",
  "Chemin Vert (Rue du)",
  "Chemin des Poissonniers (Rue du)",
  "Cheminet (Rue du)",
  "Cheux (Rue de)",
  "Cheux (Sente de)",
  "Chevaliers (Avenue des)",
  "Chèvrefeuilles (Rue des)",
  "Chinon (Rue de)",
  "Choron (Rue)",
  "Chrestien de Troyes (Rue)",
  "Clair Soleil (Impasse)",
  "Claude Bernard (Rue)",
  "Claude Bloch (Rue)",
  "Claude Chappe (Rue)",
  "Claude Hettier de Boislambert (Allée)",
  "Clos (Rue des)",
  "Clos Beaumois (Rue du)",
  "Clos Caillet (Rue du)",
  "Clos Charmant (Rue du)",
  "Clos Herbert (Rue du)",
  "Clos Joli (Rue du)",
  "Clos de la Bergerie (Allée du)",
  "Clos de la Prairie (Allée, du)",
  "Clos des Oiseaux (Rue du)",
  "Clos des Roses (Rue du)",
  "Colas (Passage)",
  "Colchiques (Impasse des)",
  "Colleville (Rue de)",
  "Colonel Rémy (Rue du)",
  "Colonel Usher (Rue)",
  "Commandant Antoine de Touchet (Rue du)",
  "Commandant Charcot (Rue)",
  "Commandant Le Coutour (Rue)",
  "Commerce (Place du)",
  "Commodore J. H. Hallet (Rue)",
  "Compagnons (Rue des)",
  "Compagnons de Liberation (Cours)",
  "Concorde (Avenue de la)",
  "Constant Forget (Rue)",
  "Coquelicots (Impasse des)",
  "Cordeliers (Rue des)",
  "Cordes (Rue des)",
  "Cormelles (Rue de)",
  "Cormorans (Rue des)",
  "Corneau (Rue du)",
  "Cornouailles (Rue de)",
  "Costils Beaudets (Chemin des)",
  "Costils Lambalard (Chemin des)",
  "Côte de Nacre (Avenue de la)",
  "Coteaux (Chemin des)",
  "Cotentin (Rue du)",
  "Cotonnière (Rue de la)",
  "Coudriers (Rue des)",
  "Courseulles (Avenue de)",
  "Court Bouet (Rue du)",
  "Courte Delle (Rue)",
  "Courtonne (Rue de)",
  "Courts Carreaux (Rue des)",
  "Coutures (Rue des)",
  "Couvrechef",
  "Couvrechef (Rue de)",
  "Crespellière (Venelle)",
  "Creully (Avenue de)",
  "Creux au Renard (Rue du)",
  "Criquet (Venelle)",
  "Croisiers (Rue des)",
  "Croix Guérin (Avenue)",
  "Croix de Pierre (Rue de la)",
  "Cully (Rue de)",
  "Cultures (Rue des)",
  "Cussy (Impasse de)",
  "Cussy (Rue de)",
  "Damozanne (Rue)",
  "Daniel Danjon (Rue)",
  "Daniel Huet (Rue)",
  "Défense Passive (Rue de la)",
  "Delivrande (Rue de la)",
  "Demi-Lune (Place de la)",
  "Demolombe (Rue)",
  "Denise Olive (Rue)",
  "Désert (Rue du)",
  "Deslongchamps (Rue)",
  "Desmoueux (Rue)",
  "Destriers (Allée des)",
  "Devon (Rue du)",
  "Dielette (Allée de)",
  "Dinan (Rue de)",
  "Discobole (Allée du)",
  "Dives (Rue de la)",
  "Docteur Auvray (Rue du)",
  "Docteur Calmette (Rue du)",
  "Docteur G Maugeais (Rue du)",
  "Docteur Gidon (Place)",
  "Docteur Laennec (Place du)",
  "Docteur Le Rasle (Rue)",
  "Docteur Marcel Leboucher (Rue)",
  "Docteur Maurice Collin (Avenue du)",
  "Docteur Pecker (Rue)",
  "Docteur Rayer (Rue du)",
  "Docteur Roux (Rue)",
  "Docteur Thibout de la Fresnaye (Rue du)",
  "Docteur Tillaux (Rue)",
  "Dom Aubourg (Place)",
  "Domrémy (Rue)",
  "Doyen Barbeau (Rue)",
  "Doyen Morière (Rue du)",
  "Duc Richard (Rue)",
  "Duc Rollon (Impasse)",
  "Ducs de Normandie (Résidence, les)",
  "Dumont (Impasse)",
  "Dumont d'Urville (Rue)",
  "Dunois (Boulevard)",
  "Dunois (Place)",
  "Durandal (Rue)",
  "Écureuil (Rue de l')",
  "Écuyère (Impasse)",
  "Écuyère (Rue)",
  "Édimbourg (Avenue d')",
  "Edmond Boca (Rue)",
  "Edmond Gombeaux (Rue)",
  "Edmond Rostand (Rue)",
  "Edmond Villey Desmezerets (Rue)",
  "Égalité (Rue de l')",
  "Églantiers (Rue des)",
  "Église (Rue de l')",
  "Église de Vaucelles (Rue de l')",
  "Eliane (Rue)",
  "Élie de Beaumont (Rue)",
  "Emmanuel Bénard (Rue)",
  "Emmanuel Desbiot (Rue)",
  "Enchanteur Merlin (Avenue)",
  "Engannerie (Rue de l')",
  "Épargne (Rue de l')",
  "Épron (Rue d')",
  "Équipes d'Urgence (Rue des)",
  "Ernest Blot (Allée)",
  "Ernest Manchon (Rue)",
  "Escoville (Passage, d')",
  "Espérance (Boulevard de l')",
  "Etavaux (Rue d')",
  "Étrier (Rue de l')",
  "Eugène Boudin (Rue)",
  "Eugène Maes (Rue)",
  "Eugène Meslin (Quai)",
  "Eugénie (Rue)",
  "Eure (Rue de l')",
  "Europe (Porte, de l')",
  "Eustache Restout (Rue)",
  "Fabrique (Rue de la)",
  "Falaise (Rue de)",
  "Faraday (Rue)",
  "Farman (Rue)",
  "Fauvettes (Rue des)",
  "Félix Éboué (Place)",
  "Fer Paris A Cherbo (Cheminement de)",
  "Fer de Caen A Fl (Chemin de)",
  "Fer de Caen A Vi (Chemin de)",
  "Fermat (Rue)",
  "Fernand Léger (Rue)",
  "Finistère (Rue du)",
  "Finlande (Rue de)",
  "Flandre (Rue, de)",
  "Flandres Dunkerque (Avenue)",
  "Fleurs (Allée des)",
  "Fleury-sur-Orne (Chemin de)",
  "Florals (Allée des)",
  "Folie (Rue de la)",
  "Fontaine (Rue de la)",
  "Fontaine Henry (Rue de)",
  "Fontaine Venoise (Square, de la)",
  "Fontaine aux Dames (Place, de la)",
  "Fontette (Place)",
  "Formigny (Rue de)",
  "Fort (Promenade du)",
  "Fosses (Rue des)",
  "Fosses Saint Julien (Promenade des)",
  "Fossés Saint-Julien (Rue des)",
  "Fosses du Château (Rue des)",
  "Fossette (Cour de la)",
  "Four (Rue du)",
  "Fraisiers (Rue des)",
  "François Marescot (Rue)",
  "François Mitterrand (Quai)",
  "Franqueville (Rue de)",
  "Fraternité (Rue de la)",
  "Fred Scamaroni (Rue)",
  "Frémentel (Rue)",
  "Frères Boutrois (Rue des)",
  "Frères Colin (Rue des)",
  "Frères Lumière (Rue des)",
  "Frères Michaut (Rue des)",
  "Fresnel (Rue)",
  "Froide (Rue)",
  "Fromages (Rue aux)",
  "Gabriel Dupont (Rue)",
  "Gaillarde (Rue)",
  "Gaillon (Rue du)",
  "Galmanche (Rue de)",
  "Gambetta (Place)",
  "Gandhi (Rue)",
  "Gardin (Place)",
  "Gare (Place de la)",
  "Gare (Rue de la)",
  "Garenne (Rue de la)",
  "Gaston Lamy (Rue)",
  "Gaston Lavalley (Rue)",
  "Gautier (Venelle)",
  "Gémare (Rue)",
  "Général Decaen (Rue)",
  "Général Dempsey (Rue)",
  "Général Duparge (Rue du)",
  "General Eisenhower (Esplanade)",
  "Général Giraud (Rue)",
  "Général Laperrine (Avenue)",
  "Général Moulin (Rue du)",
  "Général Vanier (Boulevard)",
  "Genêts (Allée des)",
  "Genève (Rue de)",
  "Gens d'Armes (Rue des)",
  "Géo Lefèvre (Rue)",
  "Geôle (Rue de)",
  "Geôle (Rue, de)",
  "George Sand (Rue)",
  "Georges Auguste (Rue)",
  "Georges Brummel (Rue)",
  "Georges Cazin (Rue)",
  "Georges Clemenceau (Avenue)",
  "Georges Gaillard (Rue)",
  "Georges Goupy (Rue)",
  "Georges Lebret (Rue)",
  "Georges Lefrançois (Rue)",
  "Georges Pompidou (Boulevard)",
  "Girafe (Impasse de la)",
  "Girafe (Rue de la)",
  "Goury (Allée de)",
  "Graindorge (Rue)",
  "Grand Clos Saint-Germain (Rue du)",
  "Grand Clos Saint-Marc (Rue du)",
  "Grand Turc (Passage du)",
  "Grentheville (Rue)",
  "Gros Orme (Rue du)",
  "Gruchy (Rue de)",
  "Grusse (Rue)",
  "Guérinière (Rue de la)",
  "Guernesey (Rue de)",
  "Guerrière (Rue)",
  "Guilbert (Rue)",
  "Guillaume Gillet (Rue)",
  "Guillaume Trébutien (Rue)",
  "Guillaume de la Tremblaye (Rue)",
  "Guillaume le Conquérant (Rue)",
  "Gustave Flaubert (Rue)",
  "Guy de Maupassant (Rue)",
  "Hache (Rue de la)",
  "Hague (Rue de la)",
  "Haie Vigne (Chemin, de la)",
  "Haie Vigné (Rue de la)",
  "Haldot (Rue)",
  "Hamon (Rue)",
  "Harcourt (Avenue d')",
  "Harcourt (Route d')",
  "Hardouin Mansard (Rue)",
  "Hastings (Rue d')",
  "Haute (Rue)",
  "Haute Claire (Allée de)",
  "Hautes Bruyères (Rue des)",
  "Hautes Coutures (Place des)",
  "Hauts de Beaulieu (Rue des)",
  "Havre (Rue du)",
  "Haye Mariaise (Rue de la)",
  "Helen Keller (Rue)",
  "Hélène Boucher (Rue)",
  "Hennin (Rue du)",
  "Henri Becquerel (Boulevard)",
  "Henri Brunet (Rue)",
  "Henri Pigis (Allée)",
  "Henri Prentout (Rue)",
  "Henri le Veille (Rue)",
  "Henry Beaufils (Allée)",
  "Henry Chéron (Avenue)",
  "Henry Dunant (Rue)",
  "Hermanville (Rue d')",
  "Hérouville (Rue d')",
  "Hippodrome (Avenue de l')",
  "Hortensias (Allée des)",
  "Ifs (Route d')",
  "Isidore Pierre (Rue)",
  "Isigny (Rue d')",
  "Jacobins (Passage, des)",
  "Jacobins (Rue des)",
  "Jacques Durandas (Rue)",
  "Jacques Prévert (Rue)",
  "Jardin des Plantes (Venelle du)",
  "Jardins (Rue des)",
  "Jean Baptiste Colbert (Rue)",
  "Jean Daligault (Rue)",
  "Jean Eudes (Rue)",
  "Jean Gutenberg (Rue)",
  "Jean Hébert (Rue)",
  "Jean Jaurès (Rue)",
  "Jean Letellier (Place)",
  "Jean Marot (Rue)",
  "Jean Mermoz (Rue)",
  "Jean Monnet (Avenue)",
  "Jean Moulin (Boulevard)",
  "Jean Nouzille (Place)",
  "Jean Racine (Résidence)",
  "Jean Racine (Rue)",
  "Jean Romain (Rue)",
  "Jean de la Bruyère (Rue)",
  "Jean de la Varende (Rue)",
  "Jean le Hir (Rue)",
  "Jean-Jacques Rousseau (Rue)",
  "Jeanne d'Arc (Avenue)",
  "Jeanne d'Arc (Impasse)",
  "Jeanne d'Arc (Rue)",
  "Jersey (Rue de)",
  "Jo Tréhard (Esplanade)",
  "Jobourg (Allée de)",
  "Jonquilles (Rue des)",
  "Joseph Bédier (Allée)",
  "Joseph Philippon (Rue)",
  "Joyeuse (Rue)",
  "Juifs (Rue aux)",
  "Juillet (Quai de)",
  "Jules Grisez (Rue)",
  "Jules Oyer (Rue)",
  "Jules Rame (Rue)",
  "Jules Verne (Rue)",
  "Jumièges (Allée de)",
  "Justice (Place de la)",
  "Justice (Rue de la)",
  "Karl Probst (Rue)",
  "L'Embranchement",
  "La Cramaillere",
  "La Folie",
  "La Petite Hache",
  "La Planche au Pretre",
  "Lamartine (Rue)",
  "Lancelot (Rue)",
  "Lanfranc (Rue)",
  "Lantheuil (Rue de)",
  "Lapérouse (Allée)",
  "Laplace (Rue)",
  "Laumonnier (Rue)",
  "Lauriers (Rue des)",
  "Lausanne (Avenue de)",
  "Le Champ de Course",
  "Le Chateau",
  "Le Clos Caillet",
  "Le Clos Caillet de Bas",
  "Le Clos Pepin",
  "Le Notre (Rue)",
  "Le Petit Vallerent",
  "Lebailly (Rue)",
  "Lebisey (Rue de)",
  "Lechartier (Rue)",
  "Ledoux (Rue)",
  "Léon Lecornu (Rue)",
  "Léon Marcotte (Rue)",
  "Léonard de Vinci (Rue)",
  "Leroy (Boulevard)",
  "Leroy (Rue)",
  "Les Baladas",
  "Les Grieux",
  "Les Mallieres",
  "Les Vaux de la Folie",
  "Lessay (Allée de)",
  "Libération (Avenue de la)",
  "Liberté (Place de la)",
];

// Computed properties
const currentQuestion = computed(() => {
  return currentQuestionIndex.value >= 0 &&
    currentQuestionIndex.value < questions.length
    ? questions[currentQuestionIndex.value]
    : null;
});

const showFilteredStations = computed(
  () => stationInput.value.length > 0 && filteredStations.value.length > 0
);

const showFilteredStreets = computed(
  () => streetInput.value.length > 0 && filteredStreets.value.length > 0
);

const canGoBack = computed(() => questionPath.value.length > 1);

const isLastQuestion = computed(
  () => currentQuestionIndex.value === questions.length - 1
);

const progress = computed(() => {
  if (currentStep.value !== "survey") return 0;
  if (isSurveyComplete.value) return 100;
  const totalQuestions = questions.length;
  const currentQuestionNumber = currentQuestionIndex.value + 1;
  const isLastOrEnding =
    isLastQuestion.value ||
    currentQuestion.value?.options?.some((option) => option.next === "end");
  return isLastOrEnding
    ? 100
    : Math.min(Math.round((currentQuestionNumber / totalQuestions) * 100), 99);
});

const isValidCommuneSelection = computed(() => {
  return (
    selectedCommune.value.includes(" - ") || selectedCommune.value.trim() !== ""
  );
});

// Add these new methods
const filterStations = () => {
  const input = stationInput.value.toLowerCase();
  filteredStations.value = stationsList.filter((station) =>
    station.toLowerCase().includes(input)
  );
};

const filterStreets = () => {
  const input = streetInput.value.toLowerCase();
  filteredStreets.value = streetsList.filter((street) =>
    street.toLowerCase().includes(input)
  );
};

const selectStation = (station) => {
  stationInput.value = station;
  filteredStations.value = [];
};

const selectStreet = (street) => {
  streetInput.value = street;
  filteredStreets.value = [];
};

// Methods
const setEnqueteur = () => {
  if (enqueteur.value.trim() !== "") {
    currentStep.value = "start";
    isEnqueteurSaved.value = true;
  }
};

const startSurvey = () => {
  startDate.value = new Date().toLocaleTimeString("fr-FR", {
    hour: "2-digit",
    minute: "2-digit",
    second: "2-digit",
  });
  currentStep.value = "survey";
  answers.value = {};
  isSurveyComplete.value = false;
  questionPath.value = ["Q1"]; // Start with Q1
  currentQuestionIndex.value = 0;
};

const selectAnswer = (option, index) => {
  answers.value[currentQuestion.value.id] = index + 1;

  if (currentQuestion.value.id === "Q2") {
    if (index === 0) {
      // "Caen" sélectionné
      answers.value[`Q2_COMMUNE`] = "CAEN";
      answers.value["CODE_INSEE"] = "14118";
      answers.value["COMMUNE_LIBRE"] = "";
    }
  }

  if (option.next === "end") {
    finishSurvey();
  } else if (option.requiresPrecision) {
    nextQuestion(option.next);
  } else {
    nextQuestion();
  }
};

const handleFreeTextAnswer = () => {
  if (currentQuestion.value) {
    // Skip for street questions since they're handled by handleStreetSelection
    if (
      currentQuestion.value.id === "Q2a" ||
      currentQuestion.value.id === "Q2a_d" ||
      currentQuestion.value.id === "Q2a_nv"
    ) {
      return;
    }

    answers.value[currentQuestion.value.id] = freeTextAnswer.value;
    if (currentQuestionIndex.value < questions.length - 1) {
      nextQuestion();
    } else {
      finishSurvey();
    }
  }
};

const handleStationSelection = () => {
  if (selectedStation.value.trim() !== "") {
    answers.value[currentQuestion.value.id] = selectedStation.value;
    nextQuestion();
    selectedStation.value = ""; // Reset for next use
  }
};

const handleStreetSelection = () => {
  if (streetInput.value.trim() !== "") {
    const isListedStreet = streetsList.includes(streetInput.value);
    const questionId = currentQuestion.value.id;

    // Store the answer based on the question ID
    if (questionId === "Q2a") {
      answers.value["Q2a"] = streetInput.value;
      if (!isListedStreet) {
        answers.value["Q2a_CUSTOM"] = streetInput.value;
      }
    } else if (questionId === "Q2a_d") {
      answers.value["Q2a_d"] = streetInput.value;
      if (!isListedStreet) {
        answers.value["Q2a_d_CUSTOM"] = streetInput.value;
      }
    } else if (questionId === "Q2a_nv") {
      answers.value["Q2a_nv"] = streetInput.value;
      if (!isListedStreet) {
        answers.value["Q2a_nv_CUSTOM"] = streetInput.value;
      }
    }

    // Force move to next question
    const nextQuestionId = currentQuestion.value.next;
    if (nextQuestionId === "end") {
      finishSurvey();
    } else {
      const nextIndex = questions.findIndex((q) => q.id === nextQuestionId);
      if (nextIndex !== -1) {
        currentQuestionIndex.value = nextIndex;
        questionPath.value.push(nextQuestionId);
      }
    }

    // Reset the input
    streetInput.value = "";
    filteredStreets.value = [];
  }
};

// Add these watches
watch(stationInput, () => {
  filterStations();
});

watch(streetInput, () => {
  filterStreets();
});

const updateSelectedCommune = (value) => {
  selectedCommune.value = value;
};

const handleCommuneSelection = () => {
  if (selectedCommune.value.trim() !== "") {
    const parts = selectedCommune.value.split(" - ");
    const currentQuestionId = currentQuestion.value.id;

    if (currentQuestionId === 'Q6') {
      // Handle Q6 data saving
      if (parts.length === 2) {
        const [commune, codeInsee] = parts;
        answers.value['Q6_COMMUNE'] = commune;
        answers.value['Q6_CODE_INSEE'] = codeInsee;
        answers.value['Q6_COMMUNE_LIBRE'] = ""; // Clear any potential free text
      } else {
        answers.value['Q6_COMMUNE'] = ""; // Clear dropdown selection
        answers.value['Q6_CODE_INSEE'] = ""; // Clear INSEE code
        answers.value['Q6_COMMUNE_LIBRE'] = selectedCommune.value.trim(); // Set free text
      }
       // Also save the raw input to Q6 for backward compatibility or direct reference if needed
      answers.value['Q6'] = selectedCommune.value.trim();

    } else if (currentQuestionId === 'Q2_precision') {
       // Handle Q2_precision data saving (existing logic)
      const isNonPassenger = currentQuestionId.includes("nonvoyageur"); // This check might be redundant if only 'Q2_precision' lands here, but keep for safety
      const questionPrefix = isNonPassenger ? "Q2_nonvoyageur" : "Q2"; // Should resolve to 'Q2'

      if (parts.length === 2) {
        // Dropdown selection
        const [commune, codeInsee] = parts;
        answers.value[`${questionPrefix}_COMMUNE`] = commune;
        answers.value["CODE_INSEE"] = codeInsee;
        answers.value["COMMUNE_LIBRE"] = ""; // Clear COMMUNE_LIBRE
      } else {
        // Manual entry or free text
        answers.value[`${questionPrefix}_COMMUNE`] = ""; // Clear the dropdown commune
        answers.value["CODE_INSEE"] = ""; // Clear INSEE code
        answers.value["COMMUNE_LIBRE"] = selectedCommune.value.trim(); // Set COMMUNE_LIBRE
      }
        // Save the raw input to Q2_precision as well
       answers.value['Q2_precision'] = selectedCommune.value.trim();
    } else {
        // Fallback or handle other potential CommuneSelector uses if any
        console.warn("CommuneSelector used on unexpected question:", currentQuestionId);
         answers.value[currentQuestionId] = selectedCommune.value.trim(); // Generic save
    }


    nextQuestion();
  }
};

const nextQuestion = (forcedNextId = null) => {
  let nextQuestionId = forcedNextId;
  if (!nextQuestionId && currentQuestion.value) {
    if (
      currentQuestion.value.usesGareSelector ||
      currentQuestion.value.freeText
    ) {
      nextQuestionId = currentQuestion.value.next;
    } else {
      const selectedAnswer = answers.value[currentQuestion.value.id];
      if (currentQuestion.value.id === "Poste") {
        nextQuestionId = "Q1";
      } else {
        const selectedOption =
          currentQuestion.value.options[selectedAnswer - 1];
        nextQuestionId = selectedOption?.next || null;
      }
    }
  }

  if (nextQuestionId === "end") {
    finishSurvey();
  } else if (nextQuestionId) {
    const nextIndex = questions.findIndex((q) => q.id === nextQuestionId);
    if (nextIndex !== -1) {
      currentQuestionIndex.value = nextIndex;
      questionPath.value.push(nextQuestionId);
      freeTextAnswer.value = "";
      selectedCommune.value = "";
      postalCodePrefix.value = "";
    }
  }
};

const previousQuestion = () => {
  if (canGoBack.value) {
    // Remove current question from path
    const currentQuestionId = questionPath.value.pop();
    const previousQuestionId =
      questionPath.value[questionPath.value.length - 1];

    // Find indices
    const previousIndex = questions.findIndex(
      (q) => q.id === previousQuestionId
    );

    if (previousIndex !== -1) {
      // Update current question index
      currentQuestionIndex.value = previousIndex;

      // Clear current question's answers
      if (currentQuestionId) {
        // Clear main answer
        delete answers.value[currentQuestionId];

        // Clear any custom/additional fields
        delete answers.value[`${currentQuestionId}_CUSTOM`];

        // Clear special fields for commune questions
        if (currentQuestionId.includes("Q2")) {
          delete answers.value["Q2_COMMUNE"];
          delete answers.value["CODE_INSEE"];
          delete answers.value["COMMUNE_LIBRE"];
        }
      }

      // Reset all input fields
      freeTextAnswer.value = "";
      stationInput.value = "";
      streetInput.value = "";
      selectedCommune.value = "";
      postalCodePrefix.value = "";

      // Clear filtered lists
      filteredStations.value = [];
      filteredStreets.value = [];
    }
  }
};

const finishSurvey = async () => {
  isSurveyComplete.value = true;
  const now = new Date();
  let uniqueId = `CaenFlash-Error-${Date.now()}`; // Default in case getNextId fails
  try {
     uniqueId = await getNextId();
  } catch (error) {
     console.error("Error getting next ID:", error);
     // Potentially alert the user or prevent saving
     alert(`Erreur lors de la génération de l'ID: ${error.message}`);
     isSurveyComplete.value = false; // Revert completion state
     return;
  }

  // Ensure answers['Q1'] exists before proceeding
  const q1Answer = answers.value['Q1'];
  if (q1Answer === undefined || q1Answer === null) {
      console.error("Q1 answer is missing when trying to finish survey.");
      alert("Une erreur est survenue. Impossible de sauvegarder le questionnaire sans réponse à Q1.");
      isSurveyComplete.value = false; // Don't show completion message
      return; // Stop execution
  }

  const isPassenger = q1Answer === 1;
  const surveyType = isPassenger ? "Voyageur" : "Non-voyageur";

  // Define the object to save clearly, with fallbacks
  const dataToSave = {
    ID_questionnaire: uniqueId,
    HEURE_DEBUT: startDate.value || "Non défini", // Fallback
    DATE: now.toLocaleDateString("fr-FR").replace(/\//g, "-"),
    JOUR: now.toLocaleDateString("fr-FR", { weekday: "long" }),
    ENQUETEUR: enqueteur.value || "Non défini", // Fallback
    HEURE_FIN: now.toLocaleTimeString("fr-FR", {
      hour: "2-digit",
      minute: "2-digit",
      second: "2-digit",
    }),
    TYPE_QUESTIONNAIRE: surveyType,
    Q1: q1Answer
  };

  console.log("Attempting to save:", dataToSave); // Log the exact data

  try {
    const docRef = await addDoc(surveyCollectionRef, dataToSave); // Get reference to check saved data later if needed
    console.log("Document saved successfully with ID:", docRef.id); // Log Firestore's assigned ID
    // Compare Firestore ID with our generated one if needed: console.log("Generated ID vs Firestore ID:", uniqueId, docRef.id);
    await getDocCount(); // Keep if admin dashboard needs it
  } catch (error) {
      console.error("Error saving document to Firestore:", error);
      alert(`Erreur lors de la sauvegarde du questionnaire: ${error.message}`);
      // Potentially revert isSurveyComplete?
      isSurveyComplete.value = false;
  }
};

const resetSurvey = () => {
  currentStep.value = "start";
  startDate.value = "";
  answers.value = {};
  currentQuestionIndex.value = 0;
  questionPath.value = ["Q1"];
  freeTextAnswer.value = "";
  isSurveyComplete.value = false;
  // Note: We don't clear selectedPoste or sessionStorage
};

const getDocCount = async () => {
  try {
    const querySnapshot = await getDocs(surveyCollectionRef);
    docCount.value = querySnapshot.size;
  } catch (error) {
    console.error("Error getting document count:", error);
  }
};

const getNextId = async () => {
  const counterDoc = await getDoc(counterDocRef);
  let counter = 1;

  if (counterDoc.exists()) {
    counter = counterDoc.data().value + 1;
  }

  await setDoc(counterDocRef, { value: counter });

  return `CaenFlash-${counter.toString().padStart(6, "0")}`;
};

// Add computed property for showing partial validation button
const showPartialValidation = computed(() => {
  if (!currentQuestion.value) return false;

  // Check if we're on the correct path based on Q1 answer
  const isPassengerPath = answers.value["Q1"] === 1;
  const isDescendedPath = answers.value["Q1"] === 2;

  // Get the relevant Q5 question ID based on the path
  const relevantQ5 = isPassengerPath ? "Q5" : isDescendedPath ? "Q5_d" : null;

  if (!relevantQ5) return false;

  // Find the index of the relevant Q5
  const q5Index = questions.findIndex((q) => q.id === relevantQ5);
  if (q5Index === -1) return false;

  // Show button if we're at or past the relevant Q5 and have valid input
  return (
    currentQuestionIndex.value >= q5Index &&
    (currentQuestion.value.id === relevantQ5
      ? stationInput.value.trim() !== ""
      : currentQuestion.value.freeText
      ? freeTextAnswer.value.trim() !== ""
      : currentQuestion.value.streetInput
      ? streetInput.value.trim() !== ""
      : true)
  );
});

// Modify handlePartialValidation to save current answer before finishing
const handlePartialValidation = async () => {
  // Save current answer based on question type
  if (currentQuestion.value) {
    if (
      currentQuestion.value.id === "Q5" ||
      currentQuestion.value.id === "Q5_d"
    ) {
      const isListedStation = stationsList.includes(stationInput.value);
      if (currentQuestion.value.id === "Q5") {
        answers.value["Q5"] = stationInput.value;
        if (!isListedStation) {
          answers.value["Q5_CUSTOM"] = stationInput.value;
        }
      } else {
        answers.value["Q5_d"] = stationInput.value;
        if (!isListedStation) {
          answers.value["Q5_d_CUSTOM"] = stationInput.value;
        }
      }
    } else if (currentQuestion.value.streetInput) {
      const isListedStreet = streetsList.includes(streetInput.value);
      const questionId = currentQuestion.value.id;
      answers.value[questionId] = streetInput.value;
      if (!isListedStreet) {
        answers.value[`${questionId}_CUSTOM`] = streetInput.value;
      }
    } else if (currentQuestion.value.freeText) {
      answers.value[currentQuestion.value.id] = freeTextAnswer.value;
    }
  }

  // Save the survey
  await finishSurvey();

  // Start a new survey immediately
  startSurvey();
};

// Lifecycle hooks
onMounted(() => {
  getDocCount();
});
</script>


<style>
/* Base styles */
body {
  background-color: #2a3b63;
  margin: 0;
  padding: 0;
  font-family: Arial, sans-serif;
}

.app-container {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
  background-color: #2a3b63;
  color: white;
}

.content-container {
  flex-grow: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 5% 0;
  width: 90%;
  max-width: 600px;
  margin: 0 auto;
  box-sizing: border-box;
  overflow-y: auto;
}

.question-container {
  width: 100%;
  margin-bottom: 30px;
}

.input-container,
.station-input-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
  position: relative;
}

.form-control {
  width: 100%;
  max-width: 400px;
  padding: 10px;
  border-radius: 5px;
  border: 1px solid white;
  background-color: #333;
  color: white;
  font-size: 16px;
  margin-bottom: 15px;
}

.btn-next,
.btn-return,
.btn-option,
.btn-pdf {
  width: 100%;
  max-width: 400px;
  color: white;
  padding: 15px;
  margin-top: 10px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-size: 16px;
}

.btn-next {
  background-color: green;
}

.btn-return {
  background-color: grey;
  margin-top: 30px;
}

.btn-option {
  background-color: #4a5a83;
  text-align: left;
}

.btn-pdf {
  background-color: #3498db;
  margin: 15px auto;
  display: block;
}

.commune-dropdown {
  position: relative;
  width: 100%;
  max-width: 400px;
  max-height: 200px;
  overflow-y: auto;
  background-color: #333;
  border: 1px solid #666;
  border-radius: 5px;
  z-index: 1000;
  margin: -10px auto 15px;
  padding: 0;
  list-style: none;
}

.commune-option {
  padding: 10px;
  cursor: pointer;
  color: white;
  border-bottom: 1px solid #444;
}

.commune-option:hover {
  background-color: #444;
}

.station-input-container {
  position: relative;
  width: 100%;
  max-width: 400px;
  margin: 0 auto;
}

.logo {
  max-width: 25%;
  height: auto;
  margin-top: 40px;
  margin-bottom: 20px;
}

.footer {
  background: linear-gradient(to right, #4c4faf, #3f51b5);
  padding: 20px;
  text-align: center;
  width: 100%;
  box-sizing: border-box;
}

.progress-bar {
  width: 100%;
  height: 10px;
  background-color: #e0e0e0;
  position: relative;
  overflow: hidden;
  margin-bottom: 20px;
}

.progress {
  height: 100%;
  background-color: #4caf50;
  transition: width 0.3s ease-in-out;
}

@media screen and (max-width: 480px) {
  .commune-dropdown {
    width: 90%;
    left: 50%;
    transform: translateX(-50%);
  }

  .form-control {
    max-width: 90%;
  }
}

.modal {
  position: fixed;
  z-index: 9999;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
  background-color: #1a1a1a;
  display: flex;
  justify-content: center;
  align-items: center;
}

.modal-content {
  width: 100%;
  height: 100%;
  position: relative;
  background-color: #1a1a1a;
}

.pdf-content {
  width: 100%;
  height: 100%;
  position: relative;
}

.pdf-content iframe {
  border: none;
  width: 100%;
  height: 100%;
  background: white;
}

.close {
  position: fixed;
  right: 20px;
  top: 20px;
  width: 40px;
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: transparent;
  border: none;
  cursor: pointer;
  z-index: 10000;
  opacity: 0.7;
  transition: opacity 0.2s;
}

.close::before,
.close::after {
  content: "";
  position: absolute;
  width: 24px;
  height: 2px;
  background-color: white;
  transform-origin: center;
}

.close::before {
  transform: rotate(45deg);
}

.close::after {
  transform: rotate(-45deg);
}

.close:hover {
  opacity: 1;
}

@media screen and (min-width: 768px) {
  .modal {
    padding: 40px;
  }

  .modal-content {
    max-width: 1200px;
    margin: 0 auto;
  }
}

.btn-partial {
  width: 100%;
  max-width: 400px;
  color: white;
  padding: 15px;
  margin-top: 10px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-size: 16px;
  background-color: #ff9800;
}
</style>
