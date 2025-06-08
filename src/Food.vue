<template>
  <div class="food-container">
    <div class="upload-section">
      <input
        type="file"
        @change="onFileChange"
        accept="image/*"
        id="fileInput"
        ref="fileInput"
        :disabled="loading"
      />
      <label
        for="fileInput"
        class="file-input-label"
        :class="{ disabled: loading }"
      >
        {{ selectedFile ? selectedFile.name : "Choose an Image" }} 📸
      </label>
      <button
        @click="uploadImage"
        :disabled="!selectedFile || loading"
        class="upload-button"
      >
        Upload & Predict 🍳
      </button>
    </div>

    <div v-if="imagePreviewUrl" class="image-preview-container">
      <img :src="imagePreviewUrl" alt="Image Preview" class="image-preview" />
    </div>

    <div v-if="loading" class="loading-spinner-container">
      <div class="loading-spinner"></div>
      <p>Processing your delicious image... 🔥</p>
    </div>

    <div v-if="error" class="error-message">⚠️ {{ error }}</div>

    <div v-if="recipeInfo && !loading" class="chat-message assistant-message">
      <div class="message-header">
        <span class="role-display">Recipe Bot 🍽️</span>
        <span v-if="recipeTimestamp" class="timestamp">{{
          recipeTimestamp
        }}</span>
      </div>
      <div class="message-content recipe-info">
        <h3>{{ recipeInfo.name }}</h3>
        <p><strong>Predicted Food:</strong> {{ recipeInfo.predicted_food }}</p>
        <h4>Ingredients:</h4>
        <ul>
          <li
            v-for="(ingredient, index) in recipeInfo.ingredients"
            :key="'ingredient-' + index"
          >
            🍅 {{ ingredient }}
          </li>
        </ul>
        <h4>Steps:</h4>
        <ol>
          <li v-for="(step, index) in recipeInfo.steps" :key="'step-' + index">
            👉 {{ step }}
          </li>
        </ol>
        <p class="calories-text">
          <strong>Calories:</strong> {{ recipeInfo.calories }} kcal
        </p>
        <h4>Nutrition:</h4>
        <div v-if="recipeInfo.nutrition" class="nutrition-info">
          <p v-for="(value, key) in recipeInfo.nutrition" :key="key">
            🥗 <strong>{{ key }}:</strong> {{ value }}
          </p>
        </div>
        <div v-else>
          <p>Nutrition information not available.</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      selectedFile: null,
      imagePreviewUrl: null,
      recipeInfo: null, // Ini akan menyimpan data yang sudah diformat untuk ditampilkan
      recipeTimestamp: null,
      loading: false,
      error: null,
    };
  },
  methods: {
    onFileChange(event) {
      const file = event.target.files[0];
      if (file) {
        // Hapus URL objek lama untuk mencegah memory leak
        if (this.imagePreviewUrl && this.imagePreviewUrl.startsWith("blob:")) {
          URL.revokeObjectURL(this.imagePreviewUrl);
        }
        this.selectedFile = file;
        this.imagePreviewUrl = URL.createObjectURL(file);
        // Reset state saat gambar baru dipilih
        this.recipeInfo = null;
        this.recipeTimestamp = null;
        this.error = null;
      }
    },
    async uploadImage() {
      if (!this.selectedFile) return;

      this.loading = true;
      this.error = null;
      this.recipeInfo = null;

      const formData = new FormData();
      formData.append("file", this.selectedFile);

      try {
        const response = await fetch(
          "https://85a4-103-151-172-39.ngrok-free.app/predict_food_class/",
          {
            method: "POST",
            body: formData,
          }
        );

        if (!response.ok) {
          const errorData = await response.json();
          throw new Error(
            errorData.detail || `HTTP error! status: ${response.status}`
          );
        }

        const data = await response.json();

        // Panggil fungsi baru untuk memformat data dari backend
        const formattedData = this.formatBackendResponse(data);

        if (formattedData) {
          this.recipeInfo = formattedData;
          this.recipeTimestamp = new Date().toLocaleTimeString([], {
            hour: "2-digit",
            minute: "2-digit",
          });
        } else {
          // Ini akan menangani jika formatBackendResponse mengembalikan null
          this.error = "Failed to process the information from the server.";
        }
      } catch (err) {
        this.error = `An error occurred: ${err.message}`;
        console.error(err);
      } finally {
        this.loading = false;
      }
    },

    // --- FUNGSI UTAMA YANG DIPERBAIKI ---
    formatBackendResponse(data) {
      if (!data || !data.llm_info) {
        this.error = "No data received from the server.";
        return null;
      }

      const { predicted_food, llm_info } = data;
      const { resep, kalori, nutrisi } = llm_info;

      if (!resep) {
        this.error = "LLM did not return sufficient recipe information.";
        return null;
      }

      // Mengubah 'daging_rendang' menjadi 'Daging Rendang'
      const foodName = predicted_food
        .replace(/_/g, " ")
        .replace(/\b\w/g, (l) => l.toUpperCase());

      // Memproses string 'resep' menjadi array langkah-langkah
      // Kita asumsikan resep dipisahkan oleh newline dan nomor.
      // Regex ini akan menghapus nomor di awal (misal: "1. ", "2. ")
      const steps = resep
        .split("\n")
        .map((step) => step.trim().replace(/^\d+\.\s*/, ""))
        .filter((step) => step);

      // Karena backend tidak memisahkan bahan dan langkah, kita akan tampilkan
      // keseluruhan resep di bagian "steps" dan mengosongkan "ingredients".
      // Ini adalah penyesuaian paling logis berdasarkan data yang ada.
      const ingredients = [
        "Lihat langkah-langkah untuk detail bahan dan bumbu.",
      ];

      return {
        predicted_food: foodName, // Menggunakan nama yang sudah diformat
        name: foodName, // Judul resep
        ingredients: ingredients, // Memberi placeholder yang informatif
        steps: steps, // Array langkah-langkah yang sudah diproses
        calories: kalori || "N/A", // Menggunakan data kalori langsung
        nutrition: this.formatNutrition(nutrisi), // Menggunakan data nutrisi
      };
    },

    formatNutrition(nutritionData) {
      // Fungsi ini sudah cukup baik, hanya perlu memastikan ia menangani string
      if (typeof nutritionData === "string") {
        return { "General Info": nutritionData };
      }
      return nutritionData || { Info: "Not available" };
    },
  },
  // --- Tidak ada perubahan di 'watch' dan 'beforeUnmount' ---
  watch: {
    selectedFile(newFile, oldFile) {
      if (
        oldFile &&
        this.imagePreviewUrl &&
        this.imagePreviewUrl.startsWith("blob:")
      ) {
        URL.revokeObjectURL(this.imagePreviewUrl);
      }
      if (newFile) {
        this.recipeInfo = null;
        this.recipeTimestamp = null;
        this.error = null;
      }
    },
  },
  beforeUnmount() {
    if (this.imagePreviewUrl && this.imagePreviewUrl.startsWith("blob:")) {
      URL.revokeObjectURL(this.imagePreviewUrl);
    }
  },
};
</script>

<style scoped>
.food-container {
  max-width: 700px;
  margin: 20px auto;
  padding: 20px;
  border: 1px solid #eee;
  border-radius: 12px;
  font-family: "Roboto", "Arial", sans-serif;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
  background-color: #f9f9f9;
}

.upload-section {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-bottom: 25px;
  padding: 20px;
  background-color: #fff;
  border-radius: 10px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
}

input[type="file"] {
  display: none;
}

.file-input-label {
  padding: 12px 25px;
  background-color: #5c67f2;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 16px;
  font-weight: 500;
  transition: background-color 0.3s ease, box-shadow 0.3s ease;
  margin-bottom: 15px;
  text-align: center;
  display: inline-block;
  min-width: 220px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.file-input-label:hover {
  background-color: #4a54e0;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.15);
}

.file-input-label.disabled {
  background-color: #b0b0b0;
  cursor: not-allowed;
  box-shadow: none;
}

.upload-button {
  padding: 12px 30px;
  background-color: #f4716f;
  border: none;
  border-radius: 8px;
  color: white;
  font-weight: 600;
  font-size: 17px;
  cursor: pointer;
  transition: background-color 0.3s ease;
  min-width: 180px;
  box-shadow: 0 2px 6px rgba(244, 113, 111, 0.7);
}

.upload-button:disabled {
  background-color: #e1a3a2;
  cursor: not-allowed;
  box-shadow: none;
}

.upload-button:hover:not(:disabled) {
  background-color: #da4a46;
  box-shadow: 0 4px 12px rgba(218, 74, 70, 0.85);
}

.image-preview-container {
  display: flex;
  justify-content: center;
  margin-bottom: 25px;
}

.image-preview {
  max-width: 100%;
  max-height: 350px;
  border-radius: 10px;
  object-fit: contain;
  box-shadow: 0 6px 18px rgba(0, 0, 0, 0.1);
  border: 1px solid #ccc;
}

.loading-spinner-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-bottom: 25px;
  color: #555;
  font-size: 16px;
  font-weight: 500;
  letter-spacing: 0.04em;
}

.loading-spinner {
  border: 6px solid #f3f3f3;
  border-top: 6px solid #5c67f2;
  border-radius: 50%;
  width: 48px;
  height: 48px;
  animation: spin 1.4s linear infinite;
  margin-bottom: 10px;
}

@keyframes spin {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}

.error-message {
  background-color: #ffeded;
  color: #a94442;
  border: 1px solid #d9534f;
  padding: 14px 20px;
  border-radius: 8px;
  text-align: center;
  font-weight: 600;
  margin-bottom: 25px;
  user-select: none;
}

.chat-message.assistant-message {
  background-color: #ffffff;
  border-radius: 12px;
  padding: 25px 30px;
  box-shadow: 0 10px 22px rgba(45, 50, 120, 0.08);
  font-size: 16px;
  line-height: 1.48;
  color: #222;
  font-weight: 500;
  max-width: 100%;
}

.message-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 18px;
}

.role-display {
  font-weight: 700;
  font-size: 20px;
  color: #395eff;
  user-select: none;
}

.timestamp {
  font-size: 14px;
  color: #999;
  font-weight: 500;
  user-select: none;
}

.recipe-info h3 {
  font-weight: 700;
  font-size: 22px;
  margin-bottom: 15px;
  text-align: center;
}

.recipe-info h4 {
  font-weight: 600;
  margin-top: 18px;
  margin-bottom: 10px;
  color: #333;
}

.recipe-info ul,
.recipe-info ol {
  margin-left: 18px;
  margin-bottom: 12px;
  color: #444;
}

.recipe-info li {
  margin-bottom: 8px;
}

.calories-text {
  margin-top: 10px;
  font-weight: 600;
  color: #e94f37;
}

.nutrition-info p {
  margin: 4px 0;
  color: #3a6f3a;
  font-weight: 500;
}

/* Override text align for recipe content except the title */
.recipe-info p,
.recipe-info ul,
.recipe-info ol,
.recipe-info li,
.calories-text,
.nutrition-info p {
  text-align: left;
}
</style>
