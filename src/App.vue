<template>
  <div id="app">
    <h1>Gestion des documents</h1>

    <!-- Notifications -->
    <div
        v-if="notificationMessage"
        class="notification"
        :class="{ success: notificationType === 'success', error: notificationType === 'error' }"
    >
      {{ notificationMessage }}
    </div>

    <!-- Recherche -->
    <div class="search-container">
      <input
          v-model="searchQuery"
          placeholder="Rechercher par titre..."
          class="search-input"
      />
      <button class="btn search-btn" @click="searchDocuments">Rechercher</button>
      <button
          v-if="searchQuery"
          class="btn clear-search"
          @click="clearSearch"
      >
        Retour
      </button>
    </div>

    <!-- Liste des documents -->
    <h2>Documents enregistrés</h2>
    <div class="document-list">
      <div v-for="doc in filteredDocuments" :key="doc._id" class="document-card">
        <!-- Edition inline -->
        <div v-if="doc._id === editingDocumentId">
          <input v-model="doc.title" class="edit-input" placeholder="Titre du document" />
          <textarea v-model="doc.content" class="edit-textarea" placeholder="Contenu du document"></textarea>
          <!-- Ajout de fichier -->
          <input type="file" @change="handleEditFileUpload($event, doc)" multiple />
          <div class="actions">
            <button class="btn save" @click="saveEditedDocument(doc)">✅ Enregistrer</button>
            <button class="btn cancel" @click="cancelEdit">❌ Annuler</button>
          </div>
        </div>
        <!-- Affichage normal -->
        <div v-else>
          <h3>{{ doc.title || "Sans titre" }}</h3>
          <p>{{ doc.content || "Pas de contenu" }}</p>
          <ul v-if="doc._attachments">
            <li v-for="(file, fileName) in doc._attachments" :key="fileName">
              <a :href="getAttachmentUrl(doc._id, fileName)" target="_blank">
                {{ fileName }}
              </a>
              <button class="btn delete" @click="deleteAttachment(doc, fileName)">❌ Supprimer fichier</button>
            </li>
          </ul>
          <div class="actions">
            <button class="btn view" @click="viewDocument(doc)">👁️ Voir</button>
            <button class="btn edit" @click="editDocument(doc)">✏️ Modifier</button>
            <button class="btn delete" @click="deleteDocument(doc)">🗑️ Supprimer</button>
          </div>
        </div>
      </div>
    </div>

    <!-- Section Détails du document -->
    <div v-if="selectedDocument" class="document-details">
      <h2>Détails du document</h2>
      <p><strong>Titre :</strong> {{ selectedDocument.title || "Sans titre" }}</p>
      <p><strong>Contenu :</strong> {{ selectedDocument.content || "Aucun contenu" }}</p>
      <ul v-if="selectedDocument._attachments">
        <li v-for="(file, fileName) in selectedDocument._attachments" :key="fileName">
          <a :href="getAttachmentUrl(selectedDocument._id, fileName)" target="_blank">
            {{ fileName }}
          </a>
        </li>
      </ul>
      <button class="btn close" @click="closeDetails">Fermer</button>
    </div>

    <!-- Conteneur pour l'ajout et la génération -->
    <div class="form-container">
      <div>
        <h2>Ajouter un document</h2>
        <form @submit.prevent="saveDocument" class="document-form">
          <input v-model="newDocumentTitle" placeholder="Titre du document" required />
          <input type="file" @change="handleFileUpload" multiple />
          <button class="btn save" type="submit">Ajouter</button>
        </form>
      </div>
      <div class="generate-btn-container">
        <button class="btn generate-btn" @click="generateFakeDocuments">Générer des documents</button>
      </div>
    </div>
  </div>
</template>

<script>
import PouchDB from "pouchdb";
import PouchDBFind from "pouchdb-find";

PouchDB.plugin(PouchDBFind);

const localDb = new PouchDB("ma_base_locale");
const remoteDb = new PouchDB("http://admin:taz@127.0.0.1:5984/ma_base");

export default {
  data() {
    return {
      documents: [],
      filteredDocuments: [],
      newDocumentTitle: "",
      editingDocumentId: null,
      fileData: null,
      notificationMessage: null,
      notificationType: "success",
      searchQuery: "",
      selectedDocument: null,
    };
  },
  methods: {
    fetchDocuments() {
      localDb
          .allDocs({ include_docs: true })
          .then((result) => {
            this.documents = result.rows.map((row) => row.doc);
            this.filteredDocuments = this.documents;
          })
          .catch((err) => {
            this.showNotification(
                "Erreur lors de la récupération des documents",
                "error"
            );
            console.error(err);
          });
    },
    saveDocument() {
      const newDoc = { title: this.newDocumentTitle };
      if (this.fileData) {
        newDoc._attachments = this.fileData;
      }
      localDb
          .post(newDoc)
          .then(() => {
            this.newDocumentTitle = "";
            this.fileData = null;
            this.fetchDocuments();
            this.showNotification("Document ajouté avec succès", "success");
          })
          .catch((err) => {
            this.showNotification("Erreur lors de l'ajout du document", "error");
            console.error(err);
          });
    },
    generateFakeDocuments() {
      const fakeDocuments = Array.from({ length: 20 }, (_, i) => ({
        title: `Document fake ${i + 1}`,
        content: `Ceci est le contenu du document fake ${i + 1}`,
      }));

      Promise.all(fakeDocuments.map((doc) => localDb.post(doc)))
          .then(() => {
            this.fetchDocuments();
            this.showNotification("Documents générés avec succès", "success");
          })
          .catch((err) => {
            this.showNotification(
                "Erreur lors de la génération des documents",
                "error"
            );
            console.error(err);
          });
    },
    searchDocuments() {
      const query = this.searchQuery.trim().toLowerCase();
      if (!query) {
        this.filteredDocuments = this.documents;
        return;
      }
      this.filteredDocuments = this.documents.filter((doc) => {
        return doc.title && doc.title.toLowerCase().includes(query);
      });
    },
    clearSearch() {
      this.searchQuery = "";
      this.filteredDocuments = this.documents;
    },
    handleFileUpload(event) {
      const files = event.target.files;
      this.fileData = {};

      for (let file of files) {
        const reader = new FileReader();
        reader.onload = () => {
          this.fileData[file.name] = {
            content_type: file.type,
            data: reader.result.split(",")[1],
          };
        };
        reader.readAsDataURL(file);
      }
    },
    handleEditFileUpload(event, doc) {
      const files = event.target.files;

      if (!doc._attachments) {
        doc._attachments = {};
      }

      for (let file of files) {
        const reader = new FileReader();
        reader.onload = () => {
          doc._attachments[file.name] = {
            content_type: file.type,
            data: reader.result.split(",")[1],
          };
        };
        reader.readAsDataURL(file);
      }
    },
    editDocument(doc) {
      this.editingDocumentId = doc._id; // Activer le mode édition pour le document sélectionné
      this.fileData = null; // Réinitialiser les données de fichier
    },
    saveEditedDocument(doc) {
      // Si de nouveaux fichiers ont été ajoutés, les fusionner avec les fichiers existants
      if (this.fileData) {
        if (!doc._attachments) {
          doc._attachments = {};
        }
        Object.assign(doc._attachments, this.fileData);
      }

      localDb
          .put(doc)
          .then(() => {
            this.editingDocumentId = null;
            this.fileData = null; // Réinitialiser les données de fichier après sauvegarde
            this.fetchDocuments();
            this.showNotification("Document modifié avec succès", "success");
          })
          .catch((err) => {
            this.showNotification(
                "Erreur lors de la mise à jour du document",
                "error"
            );
            console.error(err);
          });
    },
    cancelEdit() {
      this.editingDocumentId = null;
      this.fetchDocuments();
    },
    deleteDocument(doc) {
      localDb
          .remove(doc)
          .then(() => {
            this.fetchDocuments();
            this.showNotification("Document supprimé avec succès", "success");
          })
          .catch((err) => {
            this.showNotification(
                "Erreur lors de la suppression du document",
                "error"
            );
            console.error(err);
          });
    },
    deleteAttachment(doc, fileName) {
      const updatedAttachments = { ...doc._attachments };
      delete updatedAttachments[fileName];

      const updatedDoc = { ...doc, _attachments: updatedAttachments };

      localDb
          .put(updatedDoc)
          .then(() => {
            this.fetchDocuments();
            this.showNotification("Fichier supprimé avec succès", "success");
          })
          .catch((err) => {
            this.showNotification(
                "Erreur lors de la suppression du fichier",
                "error"
            );
            console.error(err);
          });
    },
    getAttachmentUrl(docId, fileName) {
      return `${remoteDb.name}/${docId}/${fileName}`;
    },
    showNotification(message, type) {
      this.notificationMessage = message;
      this.notificationType = type;
      setTimeout(() => {
        this.notificationMessage = null;
      }, 3000);
    },
    viewDocument(doc) {
      this.selectedDocument = doc;
    },
    closeDetails() {
      this.selectedDocument = null;
    },
  },
  created() {
    this.fetchDocuments();

    localDb.sync(remoteDb, { live: true, retry: true });

    localDb.changes({ live: true, include_docs: true }).on("change", () => {
      this.fetchDocuments();
    });

    localDb
        .createIndex({ index: { fields: ["title"] } })
        .then(() => {
          console.log("Index créé avec succès !");
        })
        .catch((err) => {
          console.error("Erreur lors de la création de l'index :", err);
        });
  },
};
</script>

<style>
/* Ajout pour séparer le bouton de génération */
.form-container {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 20px;
}

.generate-btn-container {
  margin-top: 30px;
}

/* Style identique pour le reste */
.notification {
  position: fixed;
  top: 20px;
  left: 50%;
  transform: translateX(-50%);
  padding: 10px 20px;
  border-radius: 5px;
  color: white;
  font-size: 16px;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
  z-index: 1000;
}

.notification.success {
  background-color: #28a745;
}

.notification.error {
  background-color: #dc3545;
}

body {
  font-family: Arial, sans-serif;
  background-color: #f7f9fc;
  margin: 0;
  padding: 20px;
}

#app {
  max-width: 800px;
  margin: auto;
  background: white;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
}

h1,
h2 {
  color: #2c3e50;
  text-align: center;
}

.document-list {
  display: flex;
  flex-wrap: wrap;
  gap: 15px;
}

.document-card {
  background: #ffffff;
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 15px;
  width: 250px;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
}

.actions {
  display: flex;
  justify-content: space-between;
  margin-top: 10px;
}

.btn.save {
  background-color: #007bff;
  color: white;
}

.btn.generate-btn {
  background-color: #007bff;
  color: white;
}

/* Bouton Modifier en vert */
.btn.edit {
  background-color: #28a745;
  color: white;
}

/* Bouton Supprimer en rouge */
.btn.delete {
  background-color: #dc3545;
  color: white;
}

.search-container {
  text-align: center;
  margin-bottom: 20px;
}

.search-input {
  padding: 10px;
  width: 60%;
  border: 1px solid #ddd;
  border-radius: 5px;
  font-size: 16px;
}

.search-container {
  text-align: center;
  margin-bottom: 20px;
}

.search-input {
  padding: 10px;
  width: 60%;
  border: 1px solid #ddd;
  border-radius: 5px;
  font-size: 16px;
}

.btn.search-btn {
  background-color: #007bff;
  color: white;
  padding: 10px;
  margin-left: 10px;
  border-radius: 5px;
  cursor: pointer;
}

.clear-search {
  background-color: #ffc107;
  color: white;
  border: none;
  border-radius: 5px;
  padding: 8px 15px;
  margin-left: 10px;
  font-size: 14px;
  cursor: pointer;
  transition: background-color 0.2s;
}

.clear-search:hover {
  background-color: #e0a800;
}
</style>

